# 笔画动画代码讲解

## 文件
`stroke_animation.html` — 使用 anime.js 实现汉字"我"的笔画顺序书写动画。

---

## 整体架构

```
单个 HTML 文件，内嵌 CSS + JS
├── <head>        引入 anime.js CDN
├── <style>       样式定义
├── <body>        SVG 画布 + 控制按钮 + 笔画信息
└── <script>      笔画数据 + 动画逻辑
```

---

## 核心原理：SVG stroke 动画

动画的本质是利用 SVG 的 **stroke-dasharray** 和 **stroke-dashoffset** 两个属性：

```
┌─────────────────────────────────────────┐
│  stroke-dasharray = 路径总长度 (L)       │
│  → 产生一个刚好覆盖整条路径的"虚线段"      │
│                                          │
│  stroke-dashoffset = L                   │
│  → 虚线完全偏移到路径之外，路径不可见       │
│                                          │
│  动画: stroke-dashoffset 从 L → 0        │
│  → 虚线逐渐回归，路径被"画"出来            │
└─────────────────────────────────────────┘
```

类比：想象一条毛毯盖住了一条路，然后一点一点拉开毛毯，路就逐渐露出来了。

### 三个关键步骤（代码中 `initPaths()` 函数）

```javascript
paths.forEach(path => {
    const len = path.getTotalLength();  // ① 获取路径总长度
    path.style.strokeDasharray = len;   // ② 设一个虚线段 = 总长度
    path.style.strokeDashoffset = len;  // ③ 完全偏移 = 不可见
});
```

---

## 笔画数据结构

"我"字共 7 笔，每笔定义为一个对象：

```javascript
const STROKES = [
    { name: '撇',  color: '#e74c3c', d: 'M 350 571 Q 380 593 ...' },
    { name: '横',  color: '#e67e22', d: 'M 584 466 Q 666 485 ...' },
    { name: '竖钩', color: '#f1c40f', d: 'M 386 457 Q 387 493 ...' },
    { name: '提',  color: '#2ecc71', d: 'M 339 289 Q 254 261 ...' },
    { name: '斜钩', color: '#3498db', d: 'M 635 195 Q 690 75 ...' },
    { name: '撇',  color: '#9b59b6', d: 'M 612 245 Q 558 197 ...' },
    { name: '点',  color: '#1abc9c', d: 'M 687 669 Q 718 648 ...' },
];
```

| 字段 | 说明 |
|------|------|
| `name` | 笔画名称（撇、横、竖钩等） |
| `color` | 笔画颜色（每笔不同色，方便区分） |
| `d` | SVG path 数据（来自 [makemeahanzi](https://github.com/skishore/makemeahanzi) 开源项目） |

### SVG path 数据说明

- `M x y` — 移动画笔到 (x, y)
- `Q cx cy x y` — 二次贝塞尔曲线，(cx,cy) 是控制点，(x,y) 是终点
- `L x y` — 直线连到 (x, y)
- `Z` — 闭合路径

坐标系统：makemeahanzi 使用 1024×900 坐标系，Y 轴向上。SVG 中通过 viewBox 和 transform 翻转显示。

---

## 动画流程（`playAnimation()` 函数）

```
点击播放
  │
  ├─ resetAnimation()  ← 重置所有笔画为不可见
  │
  └─ anime.timeline()  ← 创建时间线，7 笔按顺序播放
       │
       ├─ 第1笔 撇:  strokeDashoffset [L→0], 耗时 ~600ms
       ├─ 第2笔 横:  (与前一笔重叠 200ms)
       ├─ 第3笔 竖钩: ...
       ├─ 第4笔 提:  ...
       ├─ 第5笔 斜钩: ...
       ├─ 第6笔 撇:  ...
       └─ 第7笔 点:  → 完成提示
```

### anime.js timeline 关键代码

```javascript
const tl = anime.timeline({
    easing: 'easeInOutSine',  // 缓动函数：起止慢、中间快
    duration: 600,
});

paths.forEach((path, i) => {
    tl.add({
        targets: path,
        strokeDashoffset: [len, 0],     // 从隐藏到完全显示
        duration: 500 + len * 0.3,       // 笔画越长，时间越长
        begin: () => { /* 更新笔画名称和进度点 */ },
        complete: () => { /* 最后一笔完成提示 */ }
    }, `-=${i === 0 ? 0 : 200}`);        // 200ms 重叠，连贯书写感
});
```

- `strokeDashoffset: [len, 0]` — 数组形式表示 [起始值, 结束值]
- `duration: 500 + len * 0.3` — 动态计算，长笔画耗时更长
- `-=${200}` — 时间偏移，让相邻笔画有 200ms 重叠，看起来更流畅

---

## 页面元素说明

### SVG 画布
```html
<svg viewBox="-20 -80 1060 980">
    <g id="strokeGroup">
        <!-- JS 动态生成 7 个 <path> -->
    </g>
</svg>
```
- `viewBox="-20 -80 1060 980"` — 预留负坐标空间（部分笔画 Y 值为负）
- 每个笔画是一个独立的 `<path>` 元素

### 进度点
```html
<div class="stroke-dots" id="strokeDots">
    <!-- JS 生成 7 个圆形数字指示器 -->
</div>
```
- 播放时当前笔画放大 + 变色
- 已完成笔画变为对应颜色但缩小

### 笔画信息
实时显示「第 X / 7 笔 — 笔画名」，完成后显示「✅ 书写完成」

---

## 依赖

| 库 | 版本 | 来源 | 用途 |
|----|------|------|------|
| anime.js | 3.2.2 | CDN | SVG 动画引擎 |

无需安装，CDN 直接引入：
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/animejs/3.2.2/anime.min.js"></script>
```

---

## 如何扩展到其他字

1. 从 [makemeahanzi](https://github.com/skishore/makemeahanzi) 的 `graphics.txt` 找到目标字的 SVG path 数据
2. 按 Unicode 编码查找（如 "我" = U+6211）
3. 将每笔 path 添加到 STROKES 数组
4. 调整 viewBox 和笔画名称即可
