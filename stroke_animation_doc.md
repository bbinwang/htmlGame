# 笔画动画代码讲解

## 文件
`stroke_animation.html` — 纯前端实现汉字"我"的毛笔字笔画顺序书写动画。

---

## 整体架构

```
单个 HTML 文件，内嵌 CSS + JS
├── <head>        引入 anime.js CDN
├── <style>       宣纸风格 UI 样式
├── <body>
│   ├── SVG 画布（米字格 + 笔画层 + 毛笔滤镜）
│   ├── 笔画信息（实时显示第几笔）
│   ├── 进度点（7 个圆点）
│   └── 播放/重置按钮
└── <script>      笔画数据 + clipPath 方向性揭示动画
```

---

## 核心原理：SVG clipPath 多边形擦除

动画的本质是用 **SVG clipPath + 多边形顶点插值** 实现方向性揭示：

```
┌───────────────────────────────────────────────────┐
│  每个笔画是实心填充的 <path>（fill="#1a1a1a"）      │
│                                                    │
│  外层套一个 <clipPath>，内含一个 <polygon>           │
│  polygon 初始为退化的点（零面积）→ 笔画不可见         │
│                                                    │
│  动画: 多边形顶点从起点向完整矩形插值 (t: 0→1)       │
│  → 实心笔画被逐步"擦除"揭示出来                     │
│                                                    │
│  方向由起点决定：                                   │
│    横 → 左侧起始点，向右展开                        │
│    竖 → 顶部起始点，向下展开                        │
│    撇 → 右上起始点，向左下展开                      │
└───────────────────────────────────────────────────┘
```

### 为什么不用 strokeDasharray？

之前的方案用 `stroke-dasharray` + `stroke-dashoffset` 做描边动画，但有两个问题：

1. **空心字** — makemeahanzi 的 path 是笔画的**轮廓**（闭合形状），用 stroke 只能画出轮廓线
2. **方向不对** — strokeDashoffset 只能沿 path 绘制方向（即轮廓描绘顺序），无法控制为自然的书写方向（上→下、左→右）

新方案用 `fill` 渲染实心笔画，用 `clipPath` 控制揭示方向，两个问题同时解决。

---

## 坐标系与 Y 轴翻转

### makemeahanzi 坐标系

makemeahanzi 项目的笔画数据使用 **Y 轴向上** 的坐标系（原点在左下角，Y 值越大越靠上）。但 SVG 的坐标系是 **Y 轴向下**（原点在左上角）。

如果不翻转，字符会上下颠倒。

### 翻转方案

```html
<g id="strokeGroup" transform="translate(0, 1024) scale(1, -1)">
```

变换过程（从右到左应用）：
1. `scale(1, -1)` — Y 取反：`(x, y)` → `(x, -y)`
2. `translate(0, 1024)` — Y 偏移：`(x, -y)` → `(x, 1024 - y)`

结果：makemeahanzi 的 `y=800`（字符顶部）→ SVG 的 `y=224`（屏幕上部）✓

### Y 翻转对 clipPath 的影响

clipPath 使用 `clipPathUnits="objectBoundingBox"`（0-1 坐标系）。在这个坐标系中：

| objectBoundingBox | 本地坐标含义 | 屏幕显示位置 |
|:-:|:-:|:-:|
| Y=0 | 本地 Y 最小值 | **屏幕底部**（因 Y 翻转）|
| Y=1 | 本地 Y 最大值 | **屏幕顶部**（因 Y 翻转）|
| X=0 | 本地 X 最小值 | 屏幕左侧 |
| X=1 | 本地 X 最大值 | 屏幕右侧 |

这就是为什么"从上到下"的起始点在 Y=1（屏幕顶部），而不是 Y=0。

---

## calcPoints 函数详解

```javascript
function calcPoints(dir, t) {
    const ex = [[0,0],[1,0],[1,1],[0,1]]; // 终点：完整矩形
    let sx; // 起点：退化形状（零面积）
    switch(dir) {
        case 'left':         sx = [[0,0],[0,0],[0,1],[0,1]]; break;   // 左→右
        case 'top':          sx = [[0,1],[1,1],[1,1],[0,1]]; break;   // 上→下
        case 'top-right':    sx = [[1,1],[1,1],[1,1],[1,1]]; break;   // 右上→左下
        case 'bottom-left':  sx = [[0,0],[0,0],[0,0],[0,0]]; break;   // 左下→右上
        case 'top-left':     sx = [[0,1],[0,1],[0,1],[0,1]]; break;   // 左上→右下
    }
    return sx.map((s, j) =>
        `${(s[0]+(ex[j][0]-s[0])*t).toFixed(4)},${(s[1]+(ex[j][1]-s[1])*t).toFixed(4)}`
    ).join(' ');
}
```

### 参数

| 参数 | 含义 |
|------|------|
| `dir` | 揭示方向（对应笔画书写方向） |
| `t` | 进度 0→1（0=完全隐藏，1=完全显示） |

### 多边形插值示意（以"横"左→右为例）

```
t=0 时:  polygon 点全在左边（零宽度，不可见）
         (0,0) (0,0) (0,1) (0,1)

t=0.5:   多边形向右扩展到一半
         (0,0) (0.5,0) (0.5,1) (0,1)

t=1:     多边形完全覆盖 bounding box
         (0,0) (1,0) (1,1) (0,1)
```

4 个顶点从起点位置线性插值到终点位置，形成的多边形就是可见区域。

---

## 笔画数据结构

"我"字共 7 笔，每笔定义为一个对象：

```javascript
const STROKES = [
    { name: '撇',  color: '#e74c3c', d: 'M 350 571 Q ...' },
    { name: '横',  color: '#e67e22', d: 'M 584 466 Q ...' },
    // ...
];
```

| 字段 | 说明 |
|------|------|
| `name` | 笔画名称（撇、横、竖钩等） |
| `color` | 进度点颜色（区分不同笔画，实际笔画渲染为墨黑色） |
| `d` | SVG path 数据（来自 [makemeahanzi](https://github.com/skishore/makemeahanzi)） |

### 笔画方向映射

```javascript
const DIRECTIONS = [
    'top-right',     // 撇 — 右上起笔，向左下行笔
    'left',          // 横 — 左起右收
    'top',           // 竖钩 — 上起下收
    'bottom-left',   // 提 — 左下起右上收
    'top-left',      // 斜钩 — 左上起右下收
    'top-right',     // 撇 — 右上起左下收
    'top',           // 点 — 上起下收
];
```

方向与 STROKES 数组一一对应，决定每笔的揭示动画方向。

---

## 动画流程

```
点击播放
  │
  ├─ resetAnimation()     ← 隐藏所有笔画，重置 clip 多边形
  │
  └─ 逐笔 setTimeout 链   ← 每笔间隔 75% 的笔画时长
       │
       ├─ 设置 opacity=1
       ├─ 更新笔画信息和进度点
       └─ requestAnimationFrame 循环
            │
            ├─ 计算 t = (now-start)/duration
            ├─ easeInOutQuad(t) 缓动
            ├─ 更新 clip 多边形 points
            └─ t < 1 → 继续循环
               t = 1 → 下一笔 / 完成
```

### 缓动函数

```javascript
const e = t < 0.5 ? 2*t*t : -1+(4-2*t)*t; // easeInOutQuad
```

先加速后减速，模拟毛笔起笔和收笔的节奏感。

### 取消机制

```javascript
let animId = 0;
// 播放时
const myId = ++animId;
// 每帧检查
if (animId !== myId) return; // 被新的播放/重置取消
```

点击重置或再次播放时，`animId` 递增，旧的动画循环自动退出。

---

## 毛笔效果：SVG 滤镜

```html
<filter id="brushEffect" x="-5%" y="-5%" width="110%" height="110%">
    <feTurbulence type="fractalNoise" baseFrequency="0.04" numOctaves="4" seed="3"/>
    <feDisplacementMap in="SourceGraphic" in2="noise" scale="3"
                       xChannelSelector="R" yChannelSelector="G"/>
</filter>
```

| 滤镜步骤 | 作用 |
|----------|------|
| `feTurbulence` | 生成分形噪声纹理，模拟纸张/墨迹的不规则 |
| `feDisplacementMap` | 用噪声对笔画边缘做微小位移（3px），产生粗糙的毛笔边缘 |

效果：原本光滑的 SVG path 边缘变得微微粗糙，像真实墨迹在宣纸上的晕染。

---

## 宣纸背景 + 米字格

```css
background:
    radial-gradient(... rgba(210,190,160,0.15) ...),   /* 纸张纹理1 */
    radial-gradient(... rgba(180,160,130,0.1)  ...),   /* 纸张纹理2 */
    linear-gradient(135deg, #f5f0e1, #ede4cc, #f2ead4); /* 米色底色 */
```

```html
<!-- 米字格：横竖各一条 + 两条对角线 -->
<line x1="512" y1="0" x2="512" y2="1024" stroke-dasharray="8,6"/>
<line x1="0" y1="512" x2="1024" y2="512" stroke-dasharray="8,6"/>
<line ... 对角线1 ... />
<line ... 对角线2 ... />
```

边框用 `#8b7355`（木色），网格用虚线、20% 透明度，模拟书法练习纸。

---

## 每笔的 DOM 结构

每笔生成两组元素：

```
SVG 内:
├── <defs>
│   ├── <filter id="brushEffect">         毛笔滤镜
│   └── <clipPath id="clip-0">            第1笔裁剪
│       └── <polygon points="..."/>       动画驱动的多边形
│   └── <clipPath id="clip-1">            第2笔裁剪
│       └── ...
└── <g id="strokeGroup" transform="..." filter="url(#brushEffect)">
    ├── <path d="..." fill="#1a1a1a" clip-path="url(#clip-0)"/>  第1笔
    ├── <path d="..." fill="#1a1a1a" clip-path="url(#clip-1)"/>  第2笔
    └── ...

HTML 内:
<div class="stroke-dots">
    <div class="stroke-dot" id="dot-0">1</div>
    <div class="stroke-dot" id="dot-1">2</div>
    ...
</div>
```

- **path**: 实心填充，通过 clip-path 控制可见区域
- **clipPath polygon**: 顶点由 `calcPoints()` 计算，requestAnimationFrame 驱动更新
- **stroke-dot**: 纯 HTML 元素，CSS transition 驱动颜色/缩放变化

---

## 如何扩展到其他字

### 1. 获取笔画数据

从 [makemeahanzi](https://github.com/skishore/makemeahanzi) 项目的 `graphics.txt` 中查找目标字。每行格式：
```
U+XXXX\t{"strokes": ["path1", "path2", ...]}
```

### 2. 定义笔画和方向

```javascript
const STROKES = [
    { name: '横', color: '#e74c3c', d: 'M ...' },
    { name: '竖', color: '#3498db', d: 'M ...' },
    // 按笔顺添加每一笔
];

const DIRECTIONS = [
    'left',    // 横：左→右
    'top',     // 竖：上→下
    // 按笔顺定义每笔的揭示方向
];
```

### 3. 可用方向

| 方向 | 书写方向 | 适用笔画 |
|------|---------|---------|
| `left` | 左→右 | 横 |
| `top` | 上→下 | 竖、竖钩、竖弯钩、点 |
| `top-right` | 右上→左下 | 撇 |
| `top-left` | 左上→右下 | 捺、斜钩 |
| `bottom-left` | 左下→右上 | 提 |

### 4. 调整 UI

- `subtitle` 中更新笔画总数
- `完成` 提示中更新数字

---

## 依赖

| 库 | 版本 | 用途 |
|----|------|------|
| anime.js | 3.2.2 | 引入了但未实际使用（动画改用 requestAnimationFrame） |

anime.js 在早期版本用于 timeline 动画，当前版本可移除。

---

## 关键踩坑记录

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 字符上下颠倒 | makemeahanzi Y 轴向上，SVG Y 轴向下 | `transform="translate(0,1024) scale(1,-1)"` |
| 空心描边效果 | path 是笔画轮廓，stroke 只画轮廓线 | 改用 `fill` 实心填充 |
| 揭示方向不对（竖钩从下往上） | Y-flip 让 objectBoundingBox Y=0 变成屏幕底部 | 起始点用 Y=1 表示屏幕顶部 |
| innerHTML 覆盖 canvas | `innerHTML` 赋值销毁已有 DOM | 改用 `insertAdjacentHTML` |
