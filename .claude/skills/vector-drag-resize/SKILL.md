---
name: vector-drag-resize
description: >-
  Refactor drag/resize UI interactions using 2D vector math (Vec2, Rect, 
  Hadamard product, direction sign tables). Use when the user asks to implement
  or refactor draggable/resizable elements, bounding-box resize handles, or
  wants to eliminate if-chain direction logic in drag interactions.
---

# 向量化拖拽/缩放重构

用 Vec2 / Rect 向量数学替代标量 if-chain，将 N 方向 resize 逻辑压缩为一行公式。

## 核心思路

任意 handle 方向的 resize 可以用两个向量完全编码：

- **signVec** — 鼠标位移 delta 对 (w, h) 的系数：+1 增大 / -1 缩小 / 0 不变
- **posVec** — size 收缩时 position 是否跟随：1 跟随 / 0 不动

```
方向   signVec       posVec
──────────────────────────────
br    (+1, +1)      (0, 0)
tl    (-1, -1)      (1, 1)
tr    (+1, -1)      (0, 1)
bl    (-1, +1)      (1, 0)
r     (+1,  0)      (0, 0)
l     (-1,  0)      (1, 0)
b     ( 0, +1)      (0, 0)
t     ( 0, -1)      (0, 1)
```

**统一 resize 公式**（Hadamard 积 `⊙`）：

```
newSize = max(minSize, snapSize + delta ⊙ signVec)
newPos  = snapPos + (snapSize − newSize) ⊙ posVec
```

## 实现步骤

### 1. Vec2 — 不可变二维向量

```javascript
class Vec2 {
  constructor(x = 0, y = 0) { this.x = x; this.y = y; }
  add(v)   { return new Vec2(this.x + v.x, this.y + v.y); }
  sub(v)   { return new Vec2(this.x - v.x, this.y - v.y); }
  scale(s) { return new Vec2(this.x * s, this.y * s); }
  had(v)   { return new Vec2(this.x * v.x, this.y * v.y); }
  max(v)   { return new Vec2(Math.max(this.x, v.x), Math.max(this.y, v.y)); }
  round()  { return new Vec2(Math.round(this.x), Math.round(this.y)); }
  static of(x, y) { return new Vec2(x, y); }
  static ZERO = new Vec2(0, 0);
}
```

### 2. Rect — 用 Vec2 表示矩形

```javascript
class Rect {
  constructor(pos, size) { this.pos = pos; this.size = size; }
  get x() { return this.pos.x; }
  get y() { return this.pos.y; }
  get w() { return this.size.x; }
  get h() { return this.size.y; }

  moved(delta) {
    return new Rect(this.pos.add(delta).round(), this.size);
  }
  resized(delta, signVec, posVec, minSize) {
    const newSize = this.size.add(delta.had(signVec)).max(minSize);
    const newPos  = this.pos.add(this.size.sub(newSize).had(posVec));
    return new Rect(newPos.round(), newSize.round());
  }
}
```

### 3. 方向签名表

```javascript
const DIRS = {
  tl: { sign: Vec2.of(-1,-1), pos: Vec2.of(1,1) },
  tr: { sign: Vec2.of(+1,-1), pos: Vec2.of(0,1) },
  bl: { sign: Vec2.of(-1,+1), pos: Vec2.of(1,0) },
  br: { sign: Vec2.of(+1,+1), pos: Vec2.of(0,0) },
  t:  { sign: Vec2.of( 0,-1), pos: Vec2.of(0,1) },
  b:  { sign: Vec2.of( 0,+1), pos: Vec2.of(0,0) },
  l:  { sign: Vec2.of(-1, 0), pos: Vec2.of(1,0) },
  r:  { sign: Vec2.of(+1, 0), pos: Vec2.of(0,0) },
};
```

### 4. 统一指针抽象

```javascript
function ptrPos(e) {
  const p = e.touches ? e.touches[0] : e;
  return Vec2.of(p.clientX, p.clientY);
}
```

鼠标和触摸事件绑定同一套 handler，用 `ptrPos()` 统一提取坐标。

### 5. 事件处理

交互状态用一个对象表示：

```javascript
let interaction = null;
// { type: 'move'|'resize', startPt: Vec2, snap: Rect, dir?: string, handleEl?: Element }
```

move handler 核心：

```javascript
function onPointerMove(e) {
  if (!interaction) return;
  e.preventDefault();
  const delta = ptrPos(e).sub(interaction.startPt);
  if (interaction.type === 'move') {
    state = interaction.snap.moved(delta);
  } else {
    const { sign, pos } = DIRS[interaction.dir];
    state = interaction.snap.resized(delta, sign, pos, MIN_SIZE);
  }
  applyState();
}
```

## 数学推导备忘

posVec 可以从 signVec 自动推导：对每个分量，`posVec = signVec < 0 ? 1 : 0`。
但显式写出签名表更清晰、更容易扩展（比如加入对角锁定比例等约束）。

## 扩展方向

- **等比缩放**：角 handle 拖拽时，将 delta 投影到对角线方向向量 `normalize(signVec)` 上
- **网格吸附**：在公式最终结果上做 `round(pos / grid) * grid`
- **旋转支持**：将 delta 先通过旋转矩阵逆变换到局部坐标系再走同一公式
