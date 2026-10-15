# collector — 项目地图

记录项目的来龙去脉：它从哪来、跟上游是什么关系、目录怎么组织、怎么构建部署、以及相对上游改过哪些地方。面向维护者，不是面向使用者的文档。

## 一、来源与上游关系

本项目派生自 [oil-oil/selector](https://github.com/oil-oil/selector)（MIT 协议），基于其 `d88e9a6`（2026-08-25）之后的代码独立演进。

注意：它不是 GitHub 意义上的 fork —— 仓库的 `fork` 字段为 `false`，是独立仓库，只是保留了上游的提交历史和 remote。

上游仓库被保留为第二个 git remote，方便吸收后续修复：

```bash
git remote -v          # origin = 本仓库，upstream = oil-oil/selector
git fetch upstream
git merge upstream/main
```

### 尚未合入的上游提交

| 提交 | 日期 | 内容 |
|---|---|---|
| `52ea42b` | 2026-10-11 | 换用新的品牌图标（Oil 家族），新增多个 PNG，移除 `favicon.svg`，改 `index.html` 与 `scripts/build.js` 的引用 |
| `0eb020b` | 2026-10-12 | 修正图标中选择框的居中 |

两个提交的实质内容都是品牌图标资源。合入前需要判断：这些图标属于上游品牌资产，与本项目「不体现来源」的定位是否冲突。

### 上游的付费产品

上游另有一款付费浏览器扩展 **Selector Pro**（https://selector-pro.org/），与本项目没有代码依赖。本项目的 `EXT_LANDING_URL` 仍指向它的落地页，作为设置面板里的推广位。

## 二、目录结构

```
src/                  编辑器源码，构建时按顺序拼接成单文件 bookmarklet
  core.js             入口、常量、Host 能力接口、全局钩子
  selection.js        鼠标交互：点选、Shift 多选、框选
  ui.js               面板、设置、快捷键提示
  export.js           复制、Markdown 导出、截图
  prompt.js           生成给 AI 的 prompt 文本
  sharingan.js        「写轮眼」复刻报告（样式、字体、动效、React 详情等）
  context.js          元素定位：CSS 选择器、XPath、语义路径
scripts/build.js      构建脚本，把 src/*.js 拼成 base64 载荷
assets/               图标、编辑器样式、README 配图
docs/                 文档
index.html            安装页（拖拽书签）
dist/                 构建产物，由 CI 生成，不入库
```

## 三、构建与部署

```bash
npm run build          # 等价于 node scripts/build.js
```

`scripts/build.js` 把 `src/*.js` 按固定顺序拼接，base64 编码成 `payload-*.css`，并做一次 `new Function()` 语法校验 —— 任何片段有语法错误都会让构建直接失败，不会把坏掉的 bookmarklet 发出去。

推送到 `main` 后，GitHub Actions 构建 `dist/` 并部署到 GitHub Pages（https://frog-716.github.io/collector/）。

## 四、相对上游的改动

1. **多选修饰键**：固定为 **Shift**。上游是 Shift，中途曾试过 ⌘，现已回退。
2. **项目标识重命名**：项目自有的标识符统一用 `collector`（函数名、`window.__COLLECTOR_*` 全局钩子、错误类名、console 前缀、下载文件名前缀、安装页的 localStorage key、`BOOKMARKLET_URL`）。
3. **品牌素材**：README 横幅（`assets/readme/hero.svg`）文案改为 collector。

### 故意未改的部分

- `querySelector` / `querySelectorAll`：浏览器内置 API。
- `selectorText`（CSSOM 属性）、`buildSelector` / `isUniqueSelector` / `probeSelector` 等：这些名字里的 selector 指的是 **CSS 选择器**，是本项目的核心业务对象，与项目命名无关。
- `.ai-editor-*` CSS 命名空间：与 `NS` 常量绑定，改名会让已安装的 bookmarklet 缓存失效。
- `Selector Pro` 及其网址、`assets/readme/selector-pro.svg`：属于上游的独立产品。

### 一个副作用

`window.__SELECTOR_HOST__` 改名为 `window.__COLLECTOR_HOST__` 后，上游 Selector Pro 扩展的 host 注入不再被本项目识别。这是重命名带来的必然结果，代码注释里已标注。

## 五、下一步

补齐 JS 驱动动效的抓取能力，详见 [animation-roadmap.md](animation-roadmap.md)。该能力目前**未实现**。
