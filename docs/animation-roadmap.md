# 补齐 JS 动效抓取 — 路线图

当前版本能完整抓取 CSS 动效，但抓不到由 JS 代码生成的动画（GSAP、Lottie、Framer Motion、Three.js / WebGL 等）。本文档记录现状边界、三条可行路径及其落地方式。

## 一、现状边界

抓取入口在 `src/sharingan.js` 的 `getAnimationRuntimeReport()`（约 1493 行），它汇总三块：

- `collectActiveStateRows()` — 运行时状态标记：`data-step` / `data-state` / `aria-expanded` / `aria-current` / `open`，以及 `.is-active` / `.active` 节点。对分步叙事类动画有用。
- `collectCssAnimationRows()` — 遍历元素的计算样式，收集 `animation-*` 与 `transition-*` 全套参数。上限 420 个节点、80 行。
- `collectSvgAnimationRows()` — SVG 的 `<animate>` / `<animateMotion>` / `<animateTransform>` / `<set>`，上限 40 个。

另外 `getKeyframesReport()`（约 1712 行）会从 CSSOM 里把 `@keyframes` 定义原文捞出来，含同源跨样式表。

**抓不到的根本原因**：以上全部基于「读取某一刻的静态状态」（计算样式、CSSOM 规则、DOM 属性）。JS 动画的时间轴不在这些地方——它存在于 JS 闭包里的时间轴对象中，或者每一帧直接写进 canvas 像素。

`<canvas>` 的处理见 `mediaNodeReport()`（约 1401 行）：只记录 `bitmap=WxH` 和 `context=2d|webgl`，≤320×320 时附一张 `toDataURL()` 快照。拿不到绘制逻辑。

截图路径见 `src/export.js` 的 `captureViaDisplayMedia()`：`getDisplayMedia({ video: { frameRate: 1 } })` 之后只 `grabFrame()` 取一帧。**不是录屏**，没有时序信息。

## 二、路径 A：`document.getAnimations()`（性价比最高，先做这个）

`document.getAnimations()` 是标准 Web Animations API，返回页面上**所有** `Animation` 对象——包括 CSS 动画、CSS 过渡，以及任何通过 `element.animate()` 创建的动画。

关键：它返回的是活的动画对象，能直接读出关键帧和时间参数，而不只是「此刻的样式」。

```js
function collectWaapiRows(root, selected) {
  if (!document.getAnimations) return [];
  const inScope = t => t && (t === root || root.contains(t) || selected.includes(t));
  return document.getAnimations()
    .filter(a => a.effect && inScope(a.effect.target))
    .slice(0, 60)
    .map(a => {
      const t = a.effect.getTiming();
      const frames = a.effect.getKeyframes()
        .map(f => JSON.stringify(f))
        .join(", ");
      return [
        `${describeElement(a.effect.target)} play-state=${a.playState}`,
        `  duration=${t.duration} delay=${t.delay} easing=${t.easing}`,
        `  iterations=${t.iterations} direction=${t.direction} fill=${t.fill}`,
        `  keyframes: ${limitText(frames, 800, "keyframes truncated")}`,
      ].join("\n");
    });
}
```

`getKeyframes()` 返回的是**计算后**的关键帧（含插值属性），`getTiming()` 给出完整的时序参数。这两样加起来基本等于「动画的配方」。

覆盖范围：CSS 动画（与现有实现重复，可去重）、Framer Motion 走 WAAPI 的那部分、各类用 `element.animate()` 的库、CSS 滚动驱动动画（`scroll-timeline`）。

不覆盖：GSAP（默认用 rAF 直接改 style，不经过 WAAPI）、Lottie（渲染到 SVG/Canvas）、Three.js。

接入点：在 `getAnimationRuntimeReport()` 里新增一个 section 即可，报告会自动走现有的 markdown 导出通道。

## 三、路径 B：针对具体库做 hook

### GSAP

GSAP 把所有活动动画挂在全局时间轴上，注入后可以随时遍历，不存在「注入太晚」的问题：

```js
const gsap = window.gsap;
if (gsap && gsap.globalTimeline) {
  gsap.globalTimeline.getChildren(true, true, true) // tweens, timelines, nested
    .slice(0, 80)
    .forEach(tween => {
      // tween.duration(), tween.vars（含 ease / 目标值）, tween.targets()
    });
}
// 针对某个已选元素：
gsap.getTweensOf(el);
```

滚动驱动的动效走 ScrollTrigger 插件，它同样有全局注册表：

```js
if (window.ScrollTrigger) {
  ScrollTrigger.getAll().forEach(st => {
    // st.start / st.end / st.animation / st.trigger / st.scrub
  });
}
```

这一条对落地页、营销站覆盖率高——那类页面大量使用 GSAP + ScrollTrigger。

### Lottie

Lottie 的性价比最高：每个实例的 `animationData` 就是一份**完整的、可直接复用的动画 JSON**，拿到它等于拿到了整个动效源码。

问题在于时序：书签小工具是页面加载**之后**才注入的，`lottie.loadAnimation()` 早已调用完毕，hook 不到。可行的取法：

- 注入时检查 `window.lottie` / `window.bodymovin` 是否已被全局缓存；
- 用 `MutationObserver` 监听带 `lottie` 相关 class 的容器出现，再从 DOM 反查实例；
- 兜底：读 DOM 上的 `<svg>` 结构 + 检查页面里是否有 `.json` 动画资源引用。

### Framer Motion

没有全局注册表，无法枚举实例。但它的大部分动画最终落到 WAAPI 或 rAF 改 style，所以路径 A 能捞到一部分，剩下的靠路径 C 兜底。

### Three.js / WebGL

无法解析渲染逻辑。若 `window` 上恰好挂着 renderer / scene，可以抓场景图与相机参数作为参考，但动画本身只能靠路径 C。

## 四、路径 C：录屏抽帧（通用兜底）

对 WebGL、canvas 内绘制、以及所有解析不出来的情况，这是唯一可行路径。

做法：用 `MediaRecorder` + `getDisplayMedia` 录一段短时间（比如 5 秒），或按固定间隔循环 `ImageCapture.grabFrame()`。

抽帧后有两种有价值的产物：

1. **Storyboard 精灵图** — 把若干帧横向拼成一张 PNG，贴进报告，让 AI 直接看到动画过程。这是最实用的形式。
2. **帧差异表** — 用 canvas 逐帧比对相邻帧的像素差异，自动挑出「变化最大」的几帧作为关键帧，避免把静止段也塞进去。

局限要说清楚：产物是**图像**，不是可复用的代码。AI 能看懂「从左边滑入 + 淡入」，但拿不到缓动曲线的精确参数。体积也大，只能按需生成。

## 五、建议实施顺序

1. **路径 A** — 半天量级，改动集中在 `sharingan.js` 一个函数，能立刻覆盖一批 Framer Motion / WAAPI 动画。
2. **GSAP + ScrollTrigger hook** — 覆盖率提升明显，且没有时序问题。
3. **Lottie hook** — 单独做，因为要处理注入时序，但产出价值最高（完整动画 JSON）。
4. **路径 C 录屏 storyboard** — 最后做，作为兜底。需要新增 UI 交互（录制时长、抽帧数），工程量最大。

## 六、实现时注意

- **体积**：`sharingan.js` 会被 base64 内联进 bookmarklet（当前产物约 131KB base64）。新增代码要控制体积，不要引入外部依赖。
- **去重**：路径 A 会与 `collectCssAnimationRows()` 重叠，同一动画别报两遍。
- **作用域**：所有收集函数都接受 `(root, selected)` 并只处理选中范围内的节点，新增函数保持同样的签名约定。
- **上限**：现有代码对每个 section 都设了行数/节点上限（80 行、40 个等），避免报告爆炸。新增 section 沿用同样的限制策略。
- **失败要静默**：读取页面全局对象（`window.gsap` 等）必须包在 try/catch 里，任何异常都不能影响主流程。
