# Skill: Douyin Minigame

## 当触发此 Skill

当用户说"做一个小游戏"、"抖音小游戏"、"side bar"、"侧边栏"、"体力系统"、"看广告"、"激励视频"时使用。

本 Skill 强制覆盖以下五个系统：**侧边栏接入**、**广告标识绘制**、**体力系统**、**广告模块**、**场景管理**。

---

## 1. 抖音小游戏运行时原理

在写任何代码之前，必须先理解抖音小游戏的四个核心运行时特性，它们决定了后续所有架构决策。

### 1.1 沙箱环境

抖音小游戏运行在抖音 App 内的 JavaScript 沙箱中，**没有 DOM，没有 BOM，没有 Web API**。不能操作 `document`、`window`、`localStorage`。所有能力都通过抖音注入的全局 `tt` 对象访问。

### 1.2 单 Canvas 2D 渲染

游戏界面**完全靠 Canvas 2D 绘制**。通过 `tt.createCanvas()` 创建画布，通过 `ctx.getContext('2d')` 获取绘图上下文。整个游戏只有一个 Canvas，所有场景、按钮、文字都在同一个 `ctx` 上绘制，每帧清屏重绘。

### 1.3 tt API

`tt` 是抖音注入的全局对象（不是 npm 包），提供所有 Native 能力：

| 能力 | API | 用途 |
|------|-----|------|
| 屏幕信息 | `tt.getSystemInfoSync()` | 获取屏幕宽高、像素比 |
| 画布 | `tt.createCanvas()` | 创建游戏画布 |
| 存储 | `tt.getStorageSync()` / `tt.setStorageSync()` | 本地持久化存储 |
| 触摸 | `tt.onTouchStart(cb)` | 全局触摸监听 |
| 广告 | `tt.createRewardedVideoAd()` | 创建激励视频广告 |
| 侧边栏 | `tt.navigateToScene()` | 跳转到侧边栏入口 |
| 提示 | `tt.showToast()` | 弹出轻提示 |
| 振动 | `tt.vibrateShort()` | 触觉反馈 |
| 生命周期 | `tt.onHide()` / `tt.onShow()` | 后台/前台切换 |

### 1.4 模块化

使用 CommonJS `require` / `module.exports` 进行模块化。**不需要 Webpack、Vite 或任何打包工具**。抖音开发者工具内置了打包器，它会自动将多个 `js/*.js` 文件打包成一份代码。模块按文件名自动分包（`subpackages` 字段在 `game.json` 中配置，默认空数组表示不分包）。

---

## 2. 项目配置

### 2.1 project.config.json

位于项目根目录，告诉抖音开发者工具如何编译和运行项目。这是**必须手动创建**的文件：

```json
{
  "description": "游戏名称 - 抖音小游戏",
  "setting": {
    "urlCheck": false,
    "es6": true,
    "postcss": true,
    "minified": true,
    "newFeature": true
  },
  "compileType": "game",
  "appid": "ttxxxxxxxxxxxxxxxx",
  "projectname": "project-name",
  "condition": {}
}
```

**必填字段说明：**

| 字段 | 说明 |
|------|------|
| `compileType` | 必须为 `"game"`，表示这是小游戏项目 |
| `appid` | 在 [抖音开放平台](https://developer.open-douyin.com/) 注册小游戏后获得，格式为 `tt` + 16 位字符 |
| `setting.urlCheck` | 开发阶段建议设为 `false`，避免本地开发时的网络检查 |
| `setting.es6` | 设为 `true` 允许使用 ES6 语法 |

**appid 获取方式：** 访问抖音开放平台 → 开发者工具 → 创建小游戏 → 自动生成 appid。

### 2.2 game.json

位于项目根目录，配置小游戏运行时参数：

```json
{
  "deviceOrientation": "portrait",
  "showStatusBar": false,
  "networkTimeout": {
    "request": 5000,
    "connectSocket": 5000,
    "uploadFile": 5000,
    "downloadFile": 5000
  },
  "subpackages": []
}
```

**必填字段说明：**

| 字段 | 说明 |
|------|------|
| `deviceOrientation` | `"portrait"`（竖屏）或 `"landscape"`（横屏），大多数小游戏用竖屏 |
| `showStatusBar` | `false` 隐藏系统状态栏，小游戏应全屏沉浸 |
| `networkTimeout` | 各类网络请求的超时时间（毫秒），建议设置防止卡死 |
| `subpackages` | 分包配置，小项目留空数组 `[]` |

### 2.3 用开发者工具打开项目

1. 下载并安装 [抖音开发者工具](https://developer.open-douyin.com/docs/minigame/developer-tool/download/)
2. 打开工具，选择"导入项目"
3. 选择项目根目录（包含 `project.config.json` 的文件夹）
4. 工具会自动读取 `appid`，连接本地服务
5. 点击"预览"可在模拟器中运行，点击"预览"生成二维码可在真机测试

---

## 3. 基础架构：base.js

`base.js` 是整个项目的基石，所有其他模块都从它引入核心依赖。理解 `base.js` 才能理解整个项目的渲染架构。

### 3.1 全局 Canvas 创建

```js
// base.js:7-17
const systemInfo = tt.getSystemInfoSync();
const SCREEN_WIDTH = systemInfo.windowWidth;
const SCREEN_HEIGHT = systemInfo.windowHeight;
const PIXEL_RATIO = systemInfo.pixelRatio;

const canvas = tt.createCanvas();
canvas.width = SCREEN_WIDTH * PIXEL_RATIO;
canvas.height = SCREEN_HEIGHT * PIXEL_RATIO;
const ctx = canvas.getContext('2d');
ctx.scale(PIXEL_RATIO, PIXEL_RATIO);
```

**关键设计决策：**

1. **`canvas.width` / `canvas.height` 用物理像素**（`SCREEN_WIDTH * PIXEL_RATIO`），确保在高清屏上不模糊
2. **`ctx.scale(PIXEL_RATIO)` 让后续所有绘图逻辑可以用逻辑像素**，不需要手动乘像素比
3. **`ctx` 是全局唯一的 2D 上下文**，所有场景共享同一个 `ctx`，每帧清屏重绘

### 3.2 工具函数

`base.js` 导出通用的绘图和数学工具：

```js
// base.js:24-64
randomInt(min, max)      // 随机整数 [min, max]
randomFloat(min, max)    // 随机浮点数 [min, max)
lerp(a, b, t)            // 线性插值
distance(x1, y1, x2, y2)// 两点距离
roundRect(ctx, x, y, w, h, r)  // 绘制圆角矩形路径
```

`roundRect` 只创建路径不填充，需要配合 `ctx.fill()` 或 `ctx.stroke()` 使用。

### 3.3 存储管理

```js
// base.js:68-97
const Storage = {
  get(key, defaultValue) {
    try {
      const value = tt.getStorageSync(key);
      return value !== '' && value !== undefined ? value : defaultValue;
    } catch (e) {
      console.warn('[Storage] 读取失败:', key, e);
      return defaultValue;
    }
  },
  set(key, value) {
    try {
      tt.setStorageSync(key, value);
    } catch (e) {
      console.warn('[Storage] 写入失败:', key, e);
    }
  },
  remove(key) {
    try {
      tt.removeStorageSync(key);
    } catch (e) {
      console.warn('[Storage] 删除失败:', key, e);
    }
  }
};
```

`Storage` 封装了 `tt` 的存储 API，统一处理异常和默认值。**所有需要持久化的数据（最高分、体力状态等）都通过 `Storage` 读写。**

### 3.4 场景管理器（SceneManager）

```js
// base.js:101-152
const SceneManager = {
  _scenes: {},       // { name: sceneObj }
  _current: null,    // 当前活跃场景
  _next: null,       // 下一帧要切换的场景（延迟切换）

  register(name, scene) {
    this._scenes[name] = scene;
  },

  switchTo(name, data) {
    this._next = { name, data };
  },

  switchNow(name, data) {
    if (this._current && this._current.onExit) {
      this._current.onExit();
    }
    this._current = this._scenes[name];
    if (this._current && this._current.onEnter) {
      this._current.onEnter(data);
    }
  },

  update(dt) {
    if (this._next) {
      this.switchNow(this._next.name, this._next.data);
      this._next = null;
    }
    if (this._current && this._current.update) {
      this._current.update(dt);
    }
  },

  render() {
    if (this._current && this._current.render) {
      this._current.render(ctx);
    }
  },

  handleTouch(event) {
    if (this._current && this._current.onTouch) {
      this._current.onTouch(event);
    }
  }
};
```

**为什么有 `switchTo` 和 `switchNow` 两个方法：**

- `switchNow`：立即切换，调用旧场景 `onExit` 和新场景 `onEnter`。**只在游戏启动时使用**（`game.js:46`），此时没有渲染在中间态。
- `switchTo`：延迟到下一帧 `update` 中才执行 `switchNow`。**游戏运行中所有场景切换都用这个方法**，避免在 `render` 中间切换场景导致渲染中断。

### 3.5 导出接口

```js
module.exports = {
  canvas, ctx,
  SCREEN_WIDTH, SCREEN_HEIGHT, PIXEL_RATIO,
  randomInt, randomFloat, lerp, distance, roundRect,
  Storage, SceneManager
};
```

---

## 4. 入口文件：game.js

`game.js` 是抖音小游戏的入口文件（等同于传统游戏引擎的 `main` 函数）。它在页面加载时自动执行，负责初始化所有模块、启动游戏循环、绑定全局事件。

### 4.1 完整结构

```js
// game.js - 游戏入口文件

// ==================== 模块引入 ====================
const { canvas, ctx, SCREEN_WIDTH, SCREEN_HEIGHT, SceneManager } = require('./js/base');
const Sidebar = require('./js/sidebar');
const Stamina = require('./js/stamina');
const AdManager = require('./js/ad');
const HomeScene = require('./js/scene-home');
const GameScene = require('./js/scene-game');
const ResultScene = require('./js/scene-result');

// ==================== 场景注册 ====================
SceneManager.register('home', HomeScene);
SceneManager.register('game', GameScene);
SceneManager.register('result', ResultScene);

// ==================== 初始化 ====================
AdManager.init();

const sidebarReward = Sidebar.checkReward();
if (sidebarReward > 0) {
  console.log('[Game] 用户从侧边栏进入，已发放 +' + sidebarReward + ' 体力');
}

SceneManager.switchNow('home');

// ==================== 游戏主循环 ====================
let lastTime = Date.now();

function gameLoop() {
  const now = Date.now();
  const dt = Math.min((now - lastTime) / 1000, 0.1); // 限制最大 dt 防止跳帧
  lastTime = now;

  SceneManager.update(dt);
  ctx.clearRect(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT);
  SceneManager.render();

  requestAnimationFrame(gameLoop);
}

requestAnimationFrame(gameLoop);

// ==================== 触摸事件监听 ====================
tt.onTouchStart(function (event) {
  SceneManager.handleTouch(event);
});

// ==================== 生命周期 ====================
tt.onHide(function () {
  console.log('[Game] 小游戏进入后台');
});

tt.onShow(function () {
  console.log('[Game] 小游戏恢复前台');
});
```

### 4.2 初始化顺序（有依赖关系）

| 步骤 | 代码 | 原因 |
|------|------|------|
| 1 | `SceneManager.register(...)` | 必须先注册所有场景，才能切换 |
| 2 | `AdManager.init()` | 广告需要预加载，必须最先初始化 |
| 3 | `Sidebar.checkReward()` | 检测侧边栏来源并发放奖励，在进入主页前执行 |
| 4 | `SceneManager.switchNow('home')` | 用 `switchNow` 而不是 `switchTo`，因为此时还没有在游戏循环中 |
| 5 | `requestAnimationFrame(gameLoop)` | 启动渲染循环 |
| 6 | `tt.onTouchStart(...)` | 全局触摸监听，将事件分发给当前场景 |
| 7 | `tt.onHide` / `tt.onShow` | 生命周期钩子，可用于暂停/恢复游戏 |

### 4.3 游戏循环设计

```
requestAnimationFrame(gameLoop)
  ├─ 计算 dt（帧间隔秒数，上限 0.1s 防跳帧）
  ├─ SceneManager.update(dt)   ← 处理场景切换 + 调用当前场景的 update
  ├─ ctx.clearRect(...)         ← 清屏
  ├─ SceneManager.render()      ← 调用当前场景的 render（使用全局 ctx）
  └─ requestAnimationFrame(...) ← 请求下一帧
```

**`dt` 上限 0.1 秒的原因：** 如果用户切后台再回来，`requestAnimationFrame` 可能积累大量帧间隔，不加限制会导致一次 update 中游戏逻辑跳跃式执行，圆形瞬间刷新、连击瞬间失效等异常。

### 4.4 事件分发

触摸事件通过 `tt.onTouchStart` 全局监听，直接分发给 `SceneManager.handleTouch`，再由 SceneManager 转发给当前场景的 `onTouch`。**只有当前场景能接收触摸事件**，切换场景后自动切换接收者。

---

## 5. 场景架构（至少 2 个场景）

### 5.1 场景对象结构

每个场景是一个纯对象（不是类），提供五个生命周期钩子：

```js
const HomeScene = {
  _time: 0,           // 场景内累积时间（秒），用于动画
  _startBtnRect: null, // 按钮区域（触摸判定用）

  onEnter(data) { /* 场景进入时初始化 */ },
  onExit()     { /* 场景离开时清理 */ },
  update(dt)   { /* 每帧逻辑更新 */ },
  render()     { /* 每帧 Canvas 绘制 */ },
  onTouch(event) { /* 触摸事件处理 */ }
};
```

**钩子说明：**

| 钩子 | 调用时机 | 用途 |
|------|---------|------|
| `onEnter(data)` | 场景切换完成时 | 初始化状态、计算按钮位置、接收上一场景传递的数据 |
| `onExit()` | 场景切换前 | 清理定时器、停止动画、释放资源 |
| `update(dt)` | 每帧 | 更新游戏逻辑（移动、生成、倒计时等） |
| `render()` | 每帧 | Canvas 绘制（背景、UI、图形等） |
| `onTouch(event)` | 触摸时 | 判定点击了哪个按钮/图形 |

### 5.2 强制场景要求

**必须至少包含 `home`（主页）和 `game`（游戏场景）两个场景：**

```js
SceneManager.register('home', HomeScene);
SceneManager.register('game', GameScene);
```

可选的 `result`（结算）场景在游戏结束时弹出，展示分数和广告续命按钮。

### 5.3 场景间数据传递

`SceneManager.switchTo('game', { extraLife: 3 })` 的第二个参数是数据对象，在新场景的 `onEnter(data)` 中接收：

```js
// scene-game.js:73
onEnter(data) {
  const isContinue = data && data.extraLife;
  this._life = isContinue
    ? Math.min(this._life + data.extraLife, 99)
    : INITIAL_LIFE;
}
```

---

## 6. 侧边栏（复访栏）接入

### 6.1 必接能力概述

抖音侧边栏接入必须做三件事：

1. 用户首次点击"加入复访栏"时，先弹出指引弹窗说明操作步骤
2. 用户确认理解后，再跳转到侧边栏入口页
3. 用户从侧边栏进入时，发放体力奖励

### 6.2 检测侧边栏来源

```js
// sidebar.js:22-31
const Sidebar = {
  sidebarScenes: [135222, 65846, 118014, 180732],

  isFromSidebar() {
    try {
      const launchInfo = tt.getLaunchOptionsSync();
      return this.sidebarScenes.indexOf(launchInfo.scene) !== -1;
    } catch (e) {
      return false;
    }
  }
};
```

`tt.getLaunchOptionsSync()` 在游戏启动时调用一次，返回当前启动来源的 `scene` 值。如果 `scene` 在 `sidebarScenes` 列表中，说明用户是从侧边栏进入的。

**侧边栏场景值（抖音/抖音极速版/番茄小说/红果短剧）**：
- 抖音首页侧边栏-最近使用：`021036`（或 `135222` 等）
- 抖音极速版：`101036`
- 番茄小说：`181036`
- 红果短剧：`261036`

### 6.3 跳转侧边栏

```js
// sidebar.js:33-44
navigateToSidebar() {
  return new Promise((resolve, reject) => {
    if (typeof tt === 'undefined' || !tt.navigateToScene) {
      reject(new Error('环境不支持'));
      return;
    }
    tt.navigateToScene({
      scene: 'sidebar',
      success(res) { resolve(res); },
      fail(err) { reject(err); }
    });
  });
}
```

`tt.navigateToScene({ scene: 'sidebar' })` 会唤起抖音的侧边栏入口页，用户在弹窗中点击"添加到首页"即可完成添加。

### 6.4 侧边栏指引弹窗（必须有取消按钮）

当用户点击"加入复访栏"按钮时，**必须先弹出指引面板**，说明操作步骤，**面板必须有"取消"按钮**。不能直接跳转侧边栏。

```js
// sidebar.js:64-201 - showSidebarGuide(W, H)
showSidebarGuide(W, H) {
  // 返回一个 overlay 对象，包含 draw() / handleTouch(x, y) / isDone() / shouldNavigate()
  // 面板内容：
  //   - 步骤1: 点击下方按钮进入侧边栏
  //   - 步骤2: 在弹窗中点击添加到首页
  //   - 步骤3: 下次可从侧边栏快速进入
  //   - 成功加入后，复访游戏可领取 +50 体力奖励
  // 按钮：左侧灰色【取消】按钮，右侧橙色【前往侧边栏】按钮
}
```

**在主页场景的 `onTouch` 中处理 overlay 的触摸事件，优先于所有其他按钮：**

```js
// scene-home.js:295-307
onTouch(event) {
  if (this._sidebarGuideOverlay && !this._sidebarGuideOverlay.isDone()) {
    const touch = event.touches[0];
    const result = this._sidebarGuideOverlay.handleTouch(touch.clientX, touch.clientY);
    if (result && result.action === 'go') {
      this._sidebarGuideOverlay = null;
      Sidebar.navigateToSidebar().catch(() => {});
      Sidebar.markAdded();
    } else if (result && result.action === 'close') {
      this._sidebarGuideOverlay = null;
    }
    return;
  }
  // ... 其他按钮触摸逻辑
}
```

### 6.5 侧边栏复访奖励

```js
// sidebar.js:55-62
checkReward() {
  if (this.isFromSidebar() && !this.isAdded()) {
    this.markAdded();
    Stamina.addStamina(Stamina.CONFIG.sidebarStamina);
    return Stamina.CONFIG.sidebarStamina; // 返回 50
  }
  return 0;
}
```

在主页场景入口处调用，并显示奖励提示：

```js
// scene-home.js:61-64
const reward = Sidebar.checkReward();
if (reward > 0) {
  this._rewardStamina = reward;
}
```

```js
// scene-home.js:272-293 - _drawSidebarRewardToast
_drawSidebarRewardToast() {
  ctx.fillStyle = 'rgba(6,214,160,0.15)';
  roundRect(ctx, rewardX, rewardY - rewardH / 2, rewardW, rewardH, 12);
  ctx.fill();
  ctx.strokeStyle = '#06d6a0';
  roundRect(ctx, rewardX, rewardY - rewardH / 2, rewardW, rewardH, 12);
  ctx.stroke();
  ctx.fillText('🎉 从复访栏返回！+' + this._rewardStamina + '体力奖励', ...);
}
```

---

## 7. 广告标识图形（强制要求）

**所有看广告按钮必须包含广告标识图形，符合抖音审核规范。**

### 7.1 广告标识绘制工具函数

以下绘制方法提取自 `scene-result.js:216-283`，直接复用到所有广告按钮：

```js
// scene-result.js:216 - _drawAdBadge(startX, centerY)
_drawAdBadge(startX, centerY) {
  ctx.save();
  var x = startX;

  // 第1部分：AD 文字标签（橙色圆角背景 + 白色文字）
  var tagW = 26, tagH = 16, tagR = 3;
  var tagX = x, tagY = centerY - tagH / 2;
  roundRect(ctx, tagX, tagY, tagW, tagH, tagR);
  ctx.fillStyle = '#FF9800';
  ctx.fill();
  ctx.fillStyle = '#FFFFFF';
  ctx.font = 'bold 10px sans-serif';
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  ctx.fillText('AD', tagX + tagW / 2, tagY + tagH / 2 + 1);
  x += tagW + 6;

  // 第2部分：圆形播放按钮
  var playR = 9;
  var playCx = x + playR, playCy = centerY;
  ctx.beginPath();
  ctx.arc(playCx, playCy, playR, 0, Math.PI * 2);
  ctx.fillStyle = '#FF9800';
  ctx.fill();
  ctx.fillStyle = '#FFFFFF';
  ctx.beginPath();
  ctx.moveTo(playCx - 3, playCy - 4);
  ctx.lineTo(playCx - 3, playCy + 4);
  ctx.lineTo(playCx + 4, playCy);
  ctx.closePath();
  ctx.fill();
  x += playR * 2 + 6;

  // 第3部分：奖励星标（五角星 + 小红心）
  var starCx = x + 7, starCy = centerY;
  ctx.fillStyle = '#FFD700';
  this._drawStar(ctx, starCx, starCy, 5, 7, 3.5);
  ctx.fillStyle = '#FF5252';
  var heartX = starCx + 6, heartY = starCy - 6, heartSize = 3;
  // ... 绘制小红心

  ctx.restore();
}
```

### 7.2 五角星绘制工具

```js
// scene-result.js:295 - _drawStar(ctx, cx, cy, spikes, outerR, innerR)
_drawStar(ctx, cx, cy, spikes, outerR, innerR) {
  ctx.save();
  ctx.beginPath();
  var rot = Math.PI / 2 * 3;
  var step = Math.PI / spikes;
  ctx.moveTo(cx, cy - outerR);
  for (var i = 0; i < spikes; i++) {
    ctx.lineTo(cx + Math.cos(rot) * outerR, cy + Math.sin(rot) * outerR);
    rot += step;
    ctx.lineTo(cx + Math.cos(rot) * innerR, cy + Math.sin(rot) * innerR);
    rot += step;
  }
  ctx.lineTo(cx, cy - outerR);
  ctx.closePath();
  ctx.fill();
  ctx.restore();
}
```

### 7.3 使用规范

**在 `render()` 中，每个广告按钮左侧必须调用 `_drawAdBadge`：**

```js
// 示例：scene-result.js:168-175
const adBtn = this._adBtnRect;
roundRect(ctx, adBtn.x, adBtn.y, adBtn.w, adBtn.h, 23);
ctx.strokeStyle = '#FF9800';
ctx.lineWidth = 2;
ctx.stroke();
this._drawAdBadge(adBtn.x + 12, adBtn.y + adBtn.h / 2); // 广告标识
ctx.fillStyle = '#FF9800';
ctx.font = 'bold 15px sans-serif';
ctx.fillText('看广告 +3生命继续', adBtn.x + adBtn.w / 2 + 22, adBtn.y + adBtn.h / 2);
```

**广告标识三件套不可缺失：`[ AD ] [ ▶ ] [ ★ ]`，分别代表广告标签、播放按钮、奖励标识。**

---

## 8. 体力系统

### 8.1 配置

```js
// stamina.js:12-18
const CONFIG = {
  maxStamina: 120,        // 最大体力
  staminaPerMinute: 10,   // 每分钟恢复
  enterCost: 10,          // 每局消耗
  adStamina: 50,          // 看广告奖励
  sidebarStamina: 50,     // 侧边栏复访奖励
};
```

### 8.2 时间差恢复逻辑

体力恢复通过 `Date.now()` 时间差计算：

```js
// stamina.js:42-52 - recoverStamina(data)
function recoverStamina(data) {
  const now = Date.now();
  const elapsed = now - data.lastRecoverTime;
  const recoverAmount = Math.floor(elapsed / 60000) * CONFIG.staminaPerMinute;
  if (recoverAmount > 0) {
    data.stamina = Math.min(CONFIG.maxStamina, data.stamina + recoverAmount);
    data.lastRecoverTime += Math.floor(elapsed / 60000) * 60000;
    saveStaminaData(data);
  }
  return data;
}
```

**每次获取体力前先调用 `recoverStamina(data)`，确保体力已按时间恢复。**

### 8.3 持久化

```js
// stamina.js:22-40
const STORAGE_KEY = 'stamina_data_rush';

function getStaminaData() {
  try {
    const val = tt.getStorageSync(STORAGE_KEY);
    if (val) return JSON.parse(val);
  } catch (e) {}
  return { stamina: CONFIG.maxStamina, lastRecoverTime: Date.now() };
}

function saveStaminaData(data) {
  try {
    tt.setStorageSync(STORAGE_KEY, JSON.stringify(data));
  } catch (e) {}
}
```

### 8.4 公开接口

```js
const Stamina = {
  addStamina(amount) { /* 增加体力 */ },
  consumeStamina(amount) { /* 消耗体力，返回是否成功 */ },
  getCurrentStamina() { /* 获取当前体力（含恢复） */ },
  getRecoverCountdown() { /* 下次恢复倒计时(秒) */ }
};
```

### 8.5 体力条绘制

在主页场景绘制体力条，显示当前体力/最大体力，以及恢复倒计时：

```js
// scene-home.js:125-165 - _drawStaminaBar
_drawStaminaBar() {
  const stamina = Stamina.getCurrentStamina();
  const maxStamina = Stamina.CONFIG.maxStamina;
  const ratio = stamina / maxStamina;

  // 标签
  ctx.fillText('⚡ 体力', ..., ...);
  ctx.fillText(stamina + ' / ' + maxStamina, ..., ...);

  // 进度条（圆角矩形，橙色填充）
  roundRect(ctx, barX, barY, barW, barH, barH / 2);
  ctx.fillStyle = 'rgba(255,255,255,0.2)';
  ctx.fill();
  roundRect(ctx, barX, barY, barW * ratio, barH, barH / 2);
  ctx.fillStyle = ratio > 0.3 ? '#FF9800' : '#FF5252';
  ctx.fill();

  // 恢复倒计时
  const cd = Stamina.getRecoverCountdown();
  ctx.fillText('每分钟恢复10  ·  下次恢复 ' + cd + 's', ...);
}
```

---

## 9. 广告模块（激励视频）

### 9.1 初始化与生命周期

```js
// ad.js:10 - 广告单元 ID
const AD_UNIT_ID = '3vjarkp83n125604pl';

// ad.js:26-69 - init()
init() {
  this._rewardAd = tt.createRewardedVideoAd({ adUnitId: AD_UNIT_ID });
  this._rewardAd.onLoad(() => { this._isReady = true; });
  this._rewardAd.onError((err) => { this._isReady = false; });
  this._rewardAd.onClose((res) => {
    if (res && res.isEnded) {
      if (this._onRewardCallback) { this._onRewardCallback(); this._onRewardCallback = null; }
    } else {
      if (this._onCloseCallback) { this._onCloseCallback(); this._onCloseCallback = null; }
    }
  });
  this.preload();
}
```

**关键：只在 `res.isEnded === true` 时发放奖励。**

### 9.2 展示广告

```js
// ad.js:91-120 - showRewardedAd(onReward, onClose)
showRewardedAd(onReward, onClose) {
  this._onRewardCallback = onReward || null;
  this._onCloseCallback = onClose || null;
  if (this._isReady) {
    this._rewardAd.show().catch((err) => {
      this._isReady = false;
      this.preload();
    });
  } else {
    this.preload();
    tt.showToast({ title: '广告加载中，请稍候...', icon: 'none', duration: 2000 });
  }
}
```

### 9.3 典型调用方式

```js
// scene-home.js:341-351 - _watchAdForStamina
_watchAdForStamina() {
  AdManager.showRewardedAd(
    () => {
      const newStamina = Stamina.addStamina(Stamina.CONFIG.adStamina);
      tt.showToast({ title: '+' + Stamina.CONFIG.adStamina + ' 体力！当前 ' + newStamina, icon: 'success', duration: 2000 });
    },
    () => {
      tt.showToast({ title: '需完整观看广告才能获得体力', icon: 'none', duration: 2000 });
    }
  );
}
```

---

## 10. 完整文件结构

```
game.js                          # 入口：注册场景、初始化广告、启动循环
game.json                        # 小游戏配置（竖屏、超时等）
project.config.json              # 项目配置（appid、编译类型）
js/
  base.js                        # 工具函数、屏幕适配、存储、SceneManager、全局Canvas
  sidebar.js                     # 侧边栏检测、跳转、指引弹窗
  stamina.js                     # 体力系统（配置、恢复、持久化）
  ad.js                          # 激励视频广告（创建、加载、展示、回调）
  scene-home.js                  # 主页场景（体力条、开始按钮、广告按钮、复访栏按钮）
  scene-game.js                  # 游戏场景（核心玩法、生命值、连击、返回按钮）
  scene-result.js                # 结算场景（分数、广告续命按钮、再来一局）
```

---

## 11. 合规强制规则

| 规则 | 说明 |
|------|------|
| **所有广告按钮必须有广告标识** | 使用 `_drawAdBadge` 绘制 `[AD][▶][★]` 三件套 |
| **看广告前必须有确认面板** | 先弹出面板说明内容，用户点击确认后才展示广告；面板必须有取消返回按钮 |
| **侧边栏指引必须先弹窗再跳转** | 不能直接跳转，必须先展示操作指引，用户确认后才跳转 |
| **侧边栏面板必须有取消按钮** | 灰色【取消】按钮，关闭指引弹窗 |
| **至少两个场景** | 强制包含 `home`（主页）和 `game`（游戏场景） |
| **isEnded 才发奖励** | 广告关闭回调中必须检查 `res.isEnded` 再发放奖励 |
| **体力先恢复再使用** | 每次读取体力时调用 `recoverStamina`，确保按时间恢复 |

---

## 12. 发布与调试

### 12.1 真机调试流程

1. **下载开发者工具**：访问 [抖音开放平台](https://developer.open-douyin.com/) → 文档 → 开发者工具下载
2. **打开项目**：在工具中选择"导入项目"，选择包含 `project.config.json` 的根目录
3. **确认 appid**：工具会自动读取 `project.config.json` 中的 `appid`
4. **模拟器测试**：左侧模拟器可直接运行，调试 Console 输出
5. **真机测试**：点击顶部"预览"按钮，用抖音 App 扫码打开

### 12.2 广告功能调试

激励视频广告**不能在模拟器中测试**，必须在真机上测试。调试前需要在抖音开放平台后台：

1. 进入小游戏管理后台 → 广告管理 → 激励视频广告
2. 创建广告单元，获得 `adUnitId`
3. 将 `adUnitId` 填入 `js/ad.js` 的 `AD_UNIT_ID`
4. 广告单元需要审核通过后才能展示真实广告，审核期间会展示测试广告

### 12.3 侧边栏调试

侧边栏功能**不能在模拟器中完整测试**（模拟器不支持 `tt.navigateToScene`）。真机调试时：

1. 先打开游戏，点击"加入复访栏"
2. 确认指引弹窗正确显示（有取消按钮和前往按钮）
3. 点击"前往侧边栏"后，抖音会打开侧边栏入口页
4. 点击"添加到首页"完成添加
5. 返回游戏（从侧边栏或最近使用），应看到"+50 体力"奖励提示

### 12.4 上传发布

1. 在开发者工具中点击"上传"按钮
2. 填写版本号和更新说明
3. 上传后在开放平台后台提交审核
4. 审核通过后点击"发布"，游戏即可在抖音中搜索到
