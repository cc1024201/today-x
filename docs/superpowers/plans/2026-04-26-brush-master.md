# 刷猫高手 (Brush Master) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 一个单 HTML 文件的浏览器反应游戏，复刻 Brush Jjaemu，加入 4 套换肤系统和连击倍率机制。

**Architecture:** 单文件 `index.html`（内联 CSS + JS）。Canvas 层负责背景粒子动效，HTML div 层负责角色和 UI。纯逻辑函数挂在全局 `BM` 对象上（便于控制台测试）。游戏状态是单一可变对象，由 `requestAnimationFrame` + `deltaTime` 驱动。

**Tech Stack:** HTML5、CSS3、Vanilla JS、Canvas API、Pointer Events API、Web Audio API、localStorage

---

## 文件结构

```
index.html   — 游戏全部内容（HTML + CSS + JS）
```

JS 内部分段注释：
```
// ── CONSTANTS & SKINS ──────────────────────────────
// ── PURE LOGIC (testable via BM.* in console) ──────
// ── GAME STATE ─────────────────────────────────────
// ── AUDIO ──────────────────────────────────────────
// ── RENDER ─────────────────────────────────────────
// ── INPUT ──────────────────────────────────────────
// ── GAME LOOP ──────────────────────────────────────
// ── SCREENS ────────────────────────────────────────
// ── INIT ───────────────────────────────────────────
```

---

## Task 1: HTML 骨架 + CSS 变量

**Files:**
- Create: `index.html`

- [ ] **步骤 1：创建 index.html，写入完整骨架**

```html
<!DOCTYPE html>
<html lang="zh">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
  <title>刷猫高手</title>
  <style>
    :root {
      --bg: #fff8f0;
      --primary: #ff8c42;
      --accent: #ff6b35;
      --shadow: #c4622a;
      --text: #333;
      --muted: #aaa;
    }
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    html, body { width: 100%; height: 100%; overflow: hidden; background: var(--bg); font-family: sans-serif; }
    #app {
      position: relative; width: 100%; height: 100%;
      max-width: 420px; margin: 0 auto;
      display: flex; flex-direction: column;
    }
    canvas#bg-canvas {
      position: absolute; inset: 0; width: 100%; height: 100%;
      pointer-events: none; z-index: 0;
    }
    #ui { position: absolute; inset: 0; z-index: 1; }
    .screen {
      position: absolute; inset: 0;
      display: flex; flex-direction: column;
      align-items: center; justify-content: center;
      gap: 16px; padding: 24px;
    }
    .screen.hidden { display: none; }
    .btn {
      padding: 12px 32px; border-radius: 28px; border: none;
      font-size: 16px; font-weight: 700; cursor: pointer;
      transition: transform 0.1s;
    }
    .btn:active { transform: scale(0.95); }
    .btn-primary { background: var(--primary); color: #fff; box-shadow: 0 4px 0 var(--shadow); }
    .btn-secondary { background: transparent; border: 2px solid var(--primary); color: var(--primary); }
  </style>
</head>
<body>
  <div id="app">
    <canvas id="bg-canvas"></canvas>
    <div id="ui">

      <!-- ① MENU -->
      <div id="screen-menu" class="screen">
        <div id="menu-title" style="font-size:32px;font-weight:900;color:var(--accent)">刷猫高手</div>
        <div id="menu-subtitle" style="font-size:13px;color:var(--muted)">Brush Master</div>
        <div id="menu-highscore" style="font-size:13px;color:var(--muted)">最高分: 0</div>
        <button id="btn-start" class="btn btn-primary">开始游戏</button>
        <button id="btn-goto-skins" class="btn btn-secondary">选择皮肤</button>
      </div>

      <!-- ② SKIN SELECT -->
      <div id="screen-skins" class="screen hidden">
        <div style="font-size:18px;font-weight:700;color:var(--text)">选择监视者</div>
        <div id="skin-grid" style="display:grid;grid-template-columns:1fr 1fr;gap:12px;width:100%"></div>
        <button id="btn-skin-back" class="btn btn-secondary">返回</button>
      </div>

      <!-- ③ TUTORIAL -->
      <div id="screen-tutorial" class="screen hidden" style="background:rgba(0,0,0,0.85)">
        <div style="font-size:48px">🐱</div>
        <div style="font-size:20px;font-weight:700;color:#fff;text-align:center">背对你时——刷！</div>
        <div style="font-size:20px;font-weight:700;color:#ff6b35;text-align:center">转头时——停！</div>
        <div id="tutorial-timer" style="font-size:13px;color:#888">3 秒后自动开始...</div>
        <button id="btn-tutorial-skip" class="btn btn-primary" style="margin-top:8px">明白了，开始！</button>
      </div>

      <!-- ④ PLAYING -->
      <div id="screen-playing" class="screen hidden" style="justify-content:space-between;padding:16px">
        <!-- HUD -->
        <div id="hud" style="width:100%;display:flex;justify-content:space-between;align-items:center">
          <div id="score-display" style="font-size:24px;font-weight:900;color:var(--accent)">0</div>
          <div id="combo-display" style="font-size:14px;font-weight:700;color:var(--primary)"></div>
        </div>
        <!-- Character area -->
        <div id="char-area" style="flex:1;display:flex;align-items:center;justify-content:center">
          <div id="character" style="position:relative;width:120px;height:120px"></div>
        </div>
        <!-- Brush zone -->
        <div id="brush-zone"
          style="width:100%;height:100px;background:#ffe8d0;border-radius:20px;
                 display:flex;align-items:center;justify-content:center;
                 border:3px dashed #ffb880;user-select:none;touch-action:none;cursor:pointer">
          <div style="text-align:center;pointer-events:none">
            <div style="font-size:15px;font-weight:700;color:var(--primary)">按住刷毛</div>
            <div style="font-size:11px;color:var(--muted);margin-top:4px">转头立刻松手</div>
          </div>
        </div>
      </div>

      <!-- ⑤ DEAD -->
      <div id="screen-dead" class="screen hidden" style="background:#1a0a00">
        <div id="dead-char" style="font-size:64px"></div>
        <div id="dead-title" style="font-size:36px;font-weight:900;color:#ff3333;letter-spacing:3px;
             text-shadow:0 0 30px #ff000099">YOU DIED</div>
        <div id="dead-scores" style="text-align:center;color:#ff8c42;font-size:14px;line-height:2"></div>
        <button id="btn-retry" class="btn btn-primary">再来一局</button>
        <button id="btn-dead-menu" class="btn btn-secondary" style="color:#ff8c42;border-color:#ff8c42">主菜单</button>
      </div>

    </div>
  </div>
  <script>
  // ── CONSTANTS & SKINS ──────────────────────────────
  // ── PURE LOGIC ─────────────────────────────────────
  // ── GAME STATE ─────────────────────────────────────
  // ── AUDIO ──────────────────────────────────────────
  // ── RENDER ─────────────────────────────────────────
  // ── INPUT ──────────────────────────────────────────
  // ── GAME LOOP ──────────────────────────────────────
  // ── SCREENS ────────────────────────────────────────
  // ── INIT ───────────────────────────────────────────
  </script>
</body>
</html>
```

- [ ] **步骤 2：在浏览器中打开 index.html，确认主菜单可见**

用本地服务器打开（避免 ES module CORS 问题不适用，但养成好习惯）：
```bash
python3 -m http.server 8080
# 打开 http://localhost:8080
```
预期：看到橙色标题「刷猫高手」、两个按钮，背景是暖白色。

- [ ] **步骤 3：commit**

```bash
git add index.html
git commit -m "feat: html scaffold with all screen placeholders"
```

---

## Task 2: 纯逻辑函数 + 控制台测试

**Files:**
- Modify: `index.html` — `// ── PURE LOGIC ──` 段

- [ ] **步骤 1：在 `// ── PURE LOGIC ──` 后写入所有纯函数**

```javascript
// ── PURE LOGIC ─────────────────────────────────────
const BM = {}; // 挂在全局，方便控制台测试

BM.calcScore = function(brushMs, multiplier) {
  return Math.floor(brushMs * multiplier / 100);
};

BM.getMultiplier = function(comboCount) {
  if (comboCount >= 12) return 16;
  if (comboCount >= 9)  return 8;
  if (comboCount >= 6)  return 4;
  if (comboCount >= 3)  return 2;
  return 1;
};

BM.shouldDie = function(elapsedInLooking, reactionWindow, isBrushing) {
  return isBrushing && elapsedInLooking > reactionWindow;
};

BM.isFakeTurn = function(fakeTurnChance, rng) {
  return rng() < fakeTurnChance;
};

BM.getBackDuration = function(skin, rng) {
  return skin.timing.backMin + rng() * (skin.timing.backMax - skin.timing.backMin);
};

BM.getTurningDuration = function(skin, rng) {
  return skin.timing.turningMin + rng() * (skin.timing.turningMax - skin.timing.turningMin);
};
```

- [ ] **步骤 2：刷新页面，在浏览器控制台逐行运行以下断言**

```javascript
// calcScore
console.assert(BM.calcScore(3000, 1) === 30, 'calcScore 3000ms x1');
console.assert(BM.calcScore(3000, 16) === 480, 'calcScore 3000ms x16');
console.assert(BM.calcScore(0, 4) === 0, 'calcScore 0ms');

// getMultiplier
console.assert(BM.getMultiplier(0) === 1,  'multiplier 0 combo');
console.assert(BM.getMultiplier(2) === 1,  'multiplier 2 combo');
console.assert(BM.getMultiplier(3) === 2,  'multiplier 3 combo');
console.assert(BM.getMultiplier(6) === 4,  'multiplier 6 combo');
console.assert(BM.getMultiplier(9) === 8,  'multiplier 9 combo');
console.assert(BM.getMultiplier(12) === 16,'multiplier 12 combo');
console.assert(BM.getMultiplier(99) === 16,'multiplier 99 combo capped');

// shouldDie
console.assert(BM.shouldDie(600, 500, true) === true,  'shouldDie over window + brushing');
console.assert(BM.shouldDie(400, 500, true) === false, 'shouldDie within window');
console.assert(BM.shouldDie(600, 500, false) === false,'shouldDie not brushing');

// isFakeTurn
console.assert(BM.isFakeTurn(0, () => 0.5) === false, 'isFakeTurn chance=0');
console.assert(BM.isFakeTurn(1, () => 0.5) === true,  'isFakeTurn chance=1');
console.assert(BM.isFakeTurn(0.3, () => 0.29) === true,  'isFakeTurn under threshold');
console.assert(BM.isFakeTurn(0.3, () => 0.31) === false, 'isFakeTurn over threshold');
```

预期：控制台无任何 Assertion failed 输出。

- [ ] **步骤 3：commit**

```bash
git add index.html
git commit -m "feat: pure logic functions with console-verified tests"
```

---

## Task 3: 皮肤数据 + 常量定义

**Files:**
- Modify: `index.html` — `// ── CONSTANTS & SKINS ──` 段

- [ ] **步骤 1：写入常量和 4 套皮肤数据**

```javascript
// ── CONSTANTS & SKINS ──────────────────────────────
const STATE = { MENU:'MENU', SKIN_SELECT:'SKIN_SELECT', TUTORIAL:'TUTORIAL', PLAYING:'PLAYING', DEAD:'DEAD' };
const PHASE = { BACK:'BACK', TURNING:'TURNING', LOOKING:'LOOKING' };

const SKINS = [
  {
    id: 'cat',
    name: '🐱 橙猫 째무',
    emoji: '🐱',
    label: '原版手感',
    colors: { bg: '#fff8f0', primary: '#ff8c42', accent: '#ff6b35' },
    timing: { backMin: 2000, backMax: 4000, turningMin: 600, turningMax: 600, reactionWindow: 500 },
    fakeTurnChance: 0,
  },
  {
    id: 'boss',
    name: '😴 睡觉领导',
    emoji: '😴',
    label: '最难预判',
    colors: { bg: '#f3e5f5', primary: '#9c27b0', accent: '#7b1fa2' },
    timing: { backMin: 1000, backMax: 5000, turningMin: 400, turningMax: 800, reactionWindow: 500 },
    fakeTurnChance: 0,
  },
  {
    id: 'teacher',
    name: '👁️ 监考老师',
    emoji: '👁️',
    label: '假转头 · 250ms',
    colors: { bg: '#e8f5e9', primary: '#43a047', accent: '#2e7d32' },
    timing: { backMin: 1500, backMax: 3000, turningMin: 400, turningMax: 400, reactionWindow: 250 },
    fakeTurnChance: 0.3,
  },
  {
    id: 'camera',
    name: '📷 监控摄像头',
    emoji: '📷',
    label: '固定节奏',
    colors: { bg: '#e3f2fd', primary: '#1e88e5', accent: '#1565c0' },
    timing: { backMin: 2000, backMax: 2000, turningMin: 500, turningMax: 500, reactionWindow: 500 },
    fakeTurnChance: 0,
  },
];

function getSkin(id) { return SKINS.find(s => s.id === id) || SKINS[0]; }
```

- [ ] **步骤 2：控制台验证皮肤数据**

```javascript
console.assert(SKINS.length === 4, '4 skins defined');
console.assert(getSkin('cat').timing.reactionWindow === 500, 'cat reaction window');
console.assert(getSkin('teacher').timing.reactionWindow === 250, 'teacher reaction window');
console.assert(getSkin('teacher').fakeTurnChance === 0.3, 'teacher fake turn chance');
console.assert(getSkin('camera').timing.backMin === getSkin('camera').timing.backMax, 'camera fixed timing');
```

- [ ] **步骤 3：commit**

```bash
git add index.html
git commit -m "feat: skin data for all 4 characters"
```

---

## Task 4: 游戏状态对象 + 状态机切换

**Files:**
- Modify: `index.html` — `// ── GAME STATE ──` 段和 `// ── SCREENS ──` 段

- [ ] **步骤 1：写入游戏状态对象**

```javascript
// ── GAME STATE ─────────────────────────────────────
const gs = {
  screen: STATE.MENU,
  activeSkinId: 'cat',
  score: 0,
  comboCount: 0,
  maxCombo: 0,
  isBrushing: false,
  brushStartTime: 0,
  phase: PHASE.BACK,
  phaseElapsed: 0,
  phaseDuration: 2000,
  isFakeThisTurn: false,
  highScore: 0,
};
```

- [ ] **步骤 2：写入屏幕切换函数**

```javascript
// ── SCREENS ────────────────────────────────────────
const screens = {
  menu:     document.getElementById('screen-menu'),
  skins:    document.getElementById('screen-skins'),
  tutorial: document.getElementById('screen-tutorial'),
  playing:  document.getElementById('screen-playing'),
  dead:     document.getElementById('screen-dead'),
};

function showScreen(name) {
  Object.values(screens).forEach(s => s.classList.add('hidden'));
  screens[name].classList.remove('hidden');
}

function goToMenu() {
  gs.screen = STATE.MENU;
  updateHighScoreDisplay();
  showScreen('menu');
}

function goToSkins() {
  gs.screen = STATE.SKIN_SELECT;
  renderSkinGrid();
  showScreen('skins');
}

function goToPlaying() {
  gs.screen = STATE.PLAYING;
  const skin = getSkin(gs.activeSkinId);
  applyTheme(skin);
  resetPlayState(skin);
  showScreen('playing');
}

function goToDead() {
  gs.screen = STATE.DEAD;
  const skin = getSkin(gs.activeSkinId);
  document.getElementById('dead-char').textContent = skin.emoji;
  const deadScores = document.getElementById('dead-scores');
  deadScores.innerHTML =
    `本局得分 <strong>${gs.score}</strong><br>` +
    `历史最高 <strong>${gs.highScore}</strong><br>` +
    `最高连击 <strong>×${BM.getMultiplier(gs.maxCombo)}</strong>`;
  showScreen('dead');
}

function applyTheme(skin) {
  document.documentElement.style.setProperty('--bg', skin.colors.bg);
  document.documentElement.style.setProperty('--primary', skin.colors.primary);
  document.documentElement.style.setProperty('--accent', skin.colors.accent);
}

function updateHighScoreDisplay() {
  document.getElementById('menu-highscore').textContent = `最高分: ${gs.highScore}`;
}
```

- [ ] **步骤 3：写入按钮事件绑定（放在 `// ── INIT ──` 段）**

```javascript
// ── INIT ───────────────────────────────────────────
function init() {
  gs.highScore = parseInt(localStorage.getItem('brushmaster_highscore') || '0');
  updateHighScoreDisplay();

  document.getElementById('btn-start').addEventListener('click', () => {
    const tutorialSeen = localStorage.getItem('brushmaster_tutorial_seen');
    if (!tutorialSeen) { startTutorial(); } else { goToPlaying(); }
  });
  document.getElementById('btn-goto-skins').addEventListener('click', goToSkins);
  document.getElementById('btn-skin-back').addEventListener('click', goToMenu);
  document.getElementById('btn-retry').addEventListener('click', goToPlaying);
  document.getElementById('btn-dead-menu').addEventListener('click', goToMenu);

  showScreen('menu');
}

init();
```

- [ ] **步骤 4：刷新页面，验证屏幕切换**

- 点击「选择皮肤」→ 应该显示 skin_select 屏幕（当前为空白，没有卡片也没有报错）
- 点击「返回」→ 回到主菜单
- 点击「开始游戏」→ 应该跳转到 playing 屏幕（目前空白，无报错）

控制台检查无 JS 错误。

- [ ] **步骤 5：commit**

```bash
git add index.html
git commit -m "feat: game state object and screen transitions"
```

---

## Task 5: 游戏循环 (deltaTime + rAF)

**Files:**
- Modify: `index.html` — `// ── GAME LOOP ──` 和 `// ── GAME STATE ──` 段

- [ ] **步骤 1：写入 resetPlayState 和游戏循环**

```javascript
// 在 GAME STATE 段末尾追加：
function resetPlayState(skin) {
  gs.score = 0;
  gs.comboCount = 0;
  gs.maxCombo = 0;
  gs.isBrushing = false;
  gs.phase = PHASE.BACK;
  gs.phaseElapsed = 0;
  gs.phaseDuration = BM.getBackDuration(skin, Math.random);
  gs.isFakeThisTurn = false;
  document.getElementById('score-display').textContent = '0';
  document.getElementById('combo-display').textContent = '';
}

// ── GAME LOOP ─────────────────────────────────────
let lastTs = 0;

function gameLoop(ts) {
  const dt = Math.min(ts - lastTs, 100); // cap at 100ms to handle tab switching
  lastTs = ts;

  if (gs.screen === STATE.PLAYING) {
    updatePlaying(dt);
  }

  requestAnimationFrame(gameLoop);
}

function updatePlaying(dt) {
  gs.phaseElapsed += dt;
  const skin = getSkin(gs.activeSkinId);

  if (gs.phase === PHASE.BACK) {
    // 得分
    if (gs.isBrushing) {
      gs.score += BM.calcScore(dt, BM.getMultiplier(gs.comboCount));
      document.getElementById('score-display').textContent = gs.score;
    }
    // 转换
    if (gs.phaseElapsed >= gs.phaseDuration) {
      enterTurning(skin);
    }
  } else if (gs.phase === PHASE.TURNING) {
    const midpoint = gs.phaseDuration / 2;
    // 假转头：到达中点时回到 BACK
    if (gs.isFakeThisTurn && gs.phaseElapsed >= midpoint) {
      enterBack(skin);
      return;
    }
    if (gs.phaseElapsed >= gs.phaseDuration) {
      enterLooking(skin);
    }
  } else if (gs.phase === PHASE.LOOKING) {
    if (BM.shouldDie(gs.phaseElapsed, skin.timing.reactionWindow, gs.isBrushing)) {
      triggerDeath(skin);
    } else if (!gs.isBrushing && gs.phaseElapsed >= skin.timing.reactionWindow + 600) {
      // 玩家成功通过：回到 BACK
      gs.comboCount++;
      gs.maxCombo = Math.max(gs.maxCombo, gs.comboCount);
      updateComboDisplay();
      enterBack(skin);
    }
  }
}

function enterBack(skin) {
  gs.phase = PHASE.BACK;
  gs.phaseElapsed = 0;
  gs.phaseDuration = BM.getBackDuration(skin, Math.random);
  gs.isFakeThisTurn = false;
  updateCharacter();
}

function enterTurning(skin) {
  gs.phase = PHASE.TURNING;
  gs.phaseElapsed = 0;
  gs.phaseDuration = BM.getTurningDuration(skin, Math.random);
  gs.isFakeThisTurn = BM.isFakeTurn(skin.fakeTurnChance, Math.random);
  playWarningSound();
  updateCharacter();
}

function enterLooking(skin) {
  gs.phase = PHASE.LOOKING;
  gs.phaseElapsed = 0;
  gs.phaseDuration = skin.timing.reactionWindow + 600;
  updateCharacter();
}

function triggerDeath(skin) {
  gs.isBrushing = false;
  stopBrushSound();
  if (gs.score > gs.highScore) {
    gs.highScore = gs.score;
    localStorage.setItem('brushmaster_highscore', gs.highScore);
  }
  playDeathSound();
  goToDead();
}

function updateComboDisplay() {
  const mult = BM.getMultiplier(gs.comboCount);
  const el = document.getElementById('combo-display');
  if (mult > 1) {
    el.textContent = `×${mult} 连击!`;
    el.style.transform = 'scale(1.3)';
    setTimeout(() => { el.style.transform = 'scale(1)'; }, 150);
  } else {
    el.textContent = '';
  }
}
```

- [ ] **步骤 2：在 init() 中启动循环**

在 `init()` 函数最后一行（`showScreen('menu')` 之前）添加：

```javascript
requestAnimationFrame(ts => { lastTs = ts; requestAnimationFrame(gameLoop); });
```

- [ ] **步骤 3：刷新页面，开始游戏，打开控制台验证循环运行**

```javascript
// 等 2 秒后在控制台运行：
console.log(gs.phase); // 预期 'BACK' 或 'TURNING'
console.log(gs.phaseElapsed); // 预期正在增加的数字
```

- [ ] **步骤 4：commit**

```bash
git add index.html
git commit -m "feat: game loop with deltaTime and phase machine"
```

---

## Task 6: 角色渲染（橙猫 CSS）

**Files:**
- Modify: `index.html` — `<style>` 段 + `// ── RENDER ──` 段

- [ ] **步骤 1：在 `<style>` 中追加角色 CSS**

```css
/* Character */
#character { position: relative; width: 120px; height: 120px; transition: transform 0.1s; }

.char-body {
  position: absolute; width: 100px; height: 95px;
  left: 10px; top: 15px;
  border-radius: 50% 50% 45% 45%;
  background: var(--primary);
}
.char-ear-l, .char-ear-r {
  position: absolute; width: 22px; height: 28px; top: 8px;
  background: var(--primary);
  border-radius: 50% 50% 0 0;
}
.char-ear-l { left: 18px; transform: rotate(-15deg); }
.char-ear-r { right: 18px; transform: rotate(15deg); }
.char-eye-l, .char-eye-r {
  position: absolute; width: 16px; height: 18px; top: 36px;
  background: #fff; border-radius: 50%;
}
.char-eye-l { left: 28px; }
.char-eye-r { right: 28px; }
.char-pupil-l, .char-pupil-r {
  position: absolute; width: 9px; height: 10px;
  background: #222; border-radius: 50%;
  top: 4px; left: 4px;
}
.char-nose {
  position: absolute; width: 12px; height: 8px; top: 60px; left: 54px;
  background: #ffb347; border-radius: 50%;
}
.char-tail {
  position: absolute; width: 12px; height: 60px; right: 2px; top: 50px;
  background: var(--primary); border-radius: 6px;
  transform-origin: top center;
  transform: rotate(20deg);
}

/* Character states */
.char-back #character-inner { transform: scaleX(-1); } /* facing away = mirror */
.char-angry .char-eye-l, .char-angry .char-eye-r { height: 8px; top: 40px; } /* narrowed */
.char-angry .char-pupil-l, .char-pupil-r { background: #000; }
.char-dead .char-body { background: #c0392b !important; }
.char-dead .char-eye-l, .char-dead .char-eye-r { height: 4px; top: 42px; }
```

- [ ] **步骤 2：在 `// ── RENDER ──` 写入 buildCharacter 和 updateCharacter**

```javascript
// ── RENDER ─────────────────────────────────────────
function buildCharacter() {
  const el = document.getElementById('character');
  el.innerHTML = `
    <div id="character-inner" style="position:absolute;inset:0">
      <div class="char-body"></div>
      <div class="char-ear-l"></div>
      <div class="char-ear-r"></div>
      <div class="char-eye-l"><div class="char-pupil-l"></div></div>
      <div class="char-eye-r"><div class="char-pupil-r"></div></div>
      <div class="char-nose"></div>
      <div class="char-tail"></div>
    </div>`;
}

function updateCharacter() {
  const el = document.getElementById('character');
  const inner = document.getElementById('character-inner');
  el.className = '';
  if (gs.phase === PHASE.BACK) {
    inner.style.transform = 'scaleX(-1)'; // 背对
  } else if (gs.phase === PHASE.TURNING) {
    // 插值旋转：0→180 deg over phaseDuration
    const progress = Math.min(gs.phaseElapsed / gs.phaseDuration, 1);
    const deg = gs.isFakeThisTurn
      ? progress <= 0.5 ? progress * 2 * 90 : (1 - progress) * 2 * 90
      : progress * 180;
    inner.style.transform = `rotateY(${deg}deg)`;
  } else if (gs.phase === PHASE.LOOKING) {
    inner.style.transform = 'rotateY(180deg)'; // 完全面向玩家
    el.classList.add('char-angry');
  }
}

```

- [ ] **步骤 3：在 goToPlaying() 中调用 buildCharacter()**

在 `goToPlaying` 函数的 `showScreen('playing')` 之前添加：

```javascript
buildCharacter();
```

- [ ] **步骤 4：在 updatePlaying(dt) 末尾追加连续更新**

```javascript
// updatePlaying 最后一行：
if (gs.phase === PHASE.TURNING) updateCharacter();
```

- [ ] **步骤 5：刷新，开始游戏，验证**

- 进入游戏：应看到橙猫背对（mirrored）
- 等待转头：角色应有旋转动画
- 看到正脸：角色眼睛应变窄（angry 样式）

- [ ] **步骤 6：commit**

```bash
git add index.html
git commit -m "feat: orange cat CSS character with phase-driven animation"
```

---

## Task 7: Pointer Events 输入 + 刷毛交互

**Files:**
- Modify: `index.html` — `// ── INPUT ──` 段

- [ ] **步骤 1：写入输入处理**

```javascript
// ── INPUT ──────────────────────────────────────────
function setupInput() {
  const zone = document.getElementById('brush-zone');

  zone.addEventListener('pointerdown', e => {
    e.preventDefault();
    if (gs.screen !== STATE.PLAYING) return;
    gs.isBrushing = true;
    zone.style.background = '#ffd4a8';
    zone.style.transform = 'scale(0.97)';
    startBrushSound();
  });

  window.addEventListener('pointerup', e => {
    if (!gs.isBrushing) return;
    gs.isBrushing = false;
    const zone = document.getElementById('brush-zone');
    zone.style.background = '#ffe8d0';
    zone.style.transform = 'scale(1)';
    stopBrushSound();
  });

  // Prevent context menu on long press mobile
  zone.addEventListener('contextmenu', e => e.preventDefault());
}
```

- [ ] **步骤 2：在 init() 末尾（requestAnimationFrame 前）调用 setupInput()**

```javascript
setupInput();
```

- [ ] **步骤 3：刷新，开始游戏，测试输入**

- 按住刷毛区域：背景变深橙，得分应上升
- 松手：背景恢复
- 等角色转头时继续按住：应触发 YOU DIED

控制台验证：
```javascript
// 按住时：
console.log(gs.isBrushing); // true
console.log(gs.score);      // 在增加
```

- [ ] **步骤 4：commit**

```bash
git add index.html
git commit -m "feat: pointer events brush input"
```

---

## Task 8: YOU DIED 画面 + localStorage 持久化

**Files:**
- Modify: `index.html` — `// ── SCREENS ──` 段（补全 goToDead）

（goToDead 和 triggerDeath 已在 Task 5 中写入，此任务补全 YOU DIED 动画效果）

- [ ] **步骤 1：在 `<style>` 追加 YOU DIED 动画**

```css
@keyframes shake {
  0%, 100% { transform: translateX(0); }
  20%       { transform: translateX(-8px); }
  40%       { transform: translateX(8px); }
  60%       { transform: translateX(-6px); }
  80%       { transform: translateX(6px); }
}
@keyframes fadeInUp {
  from { opacity: 0; transform: translateY(20px); }
  to   { opacity: 1; transform: translateY(0); }
}
#dead-title { animation: shake 0.5s ease; }
#screen-dead .btn { animation: fadeInUp 0.4s ease 0.5s both; }
```

- [ ] **步骤 2：验证 YOU DIED 流程**

1. 开始游戏
2. 等角色转头后继续按住不松手
3. 预期：进入黑色 YOU DIED 画面，标题有抖动动画，显示本局得分/最高分/最高连击

控制台验证：
```javascript
// 死后检查 localStorage
localStorage.getItem('brushmaster_highscore'); // 应返回本局分数字符串
```

- [ ] **步骤 3：验证「再来一局」和「主菜单」按钮**

- 再来一局 → 重新进入 PLAYING，score 归零
- 主菜单 → 回到 MENU，最高分显示更新

- [ ] **步骤 4：commit**

```bash
git add index.html
git commit -m "feat: you died screen with shake animation and localStorage"
```

---

## Task 9: 皮肤选择界面

**Files:**
- Modify: `index.html` — `// ── SCREENS ──` 段（补全 renderSkinGrid）

- [ ] **步骤 1：写入 renderSkinGrid 函数**

```javascript
function renderSkinGrid() {
  const grid = document.getElementById('skin-grid');
  grid.innerHTML = '';
  SKINS.forEach(skin => {
    const card = document.createElement('div');
    card.style.cssText = `
      background: ${gs.activeSkinId === skin.id ? skin.colors.primary : '#fff'};
      color: ${gs.activeSkinId === skin.id ? '#fff' : '#333'};
      border: 3px solid ${gs.activeSkinId === skin.id ? skin.colors.accent : '#eee'};
      border-radius: 16px; padding: 16px; text-align: center; cursor: pointer;
      transition: transform 0.1s;
    `;
    card.innerHTML = `
      <div style="font-size:36px">${skin.emoji}</div>
      <div style="font-size:13px;font-weight:700;margin-top:6px">${skin.name}</div>
      <div style="font-size:11px;opacity:0.7;margin-top:2px">${skin.label}</div>
    `;
    card.addEventListener('click', () => {
      gs.activeSkinId = skin.id;
      renderSkinGrid(); // re-render to update selection
    });
    card.addEventListener('pointerdown', () => { card.style.transform = 'scale(0.95)'; });
    card.addEventListener('pointerup', () => { card.style.transform = 'scale(1)'; });
    grid.appendChild(card);
  });
}
```

- [ ] **步骤 2：刷新，点击「选择皮肤」，验证**

- 4 张皮肤卡片显示正确
- 点击任意皮肤：卡片高亮切换为该皮肤颜色
- 点击「返回」：回到主菜单
- 选一个非橙猫皮肤后开始游戏：背景颜色应变为该皮肤主题色

- [ ] **步骤 3：commit**

```bash
git add index.html
git commit -m "feat: skin selection screen with dynamic theming"
```

---

## Task 10: 其余 3 套皮肤角色视觉

**Files:**
- Modify: `index.html` — `// ── RENDER ──` 段（扩展 buildCharacter）

- [ ] **步骤 1：将 buildCharacter 改为按皮肤分发**

```javascript
function buildCharacter() {
  const el = document.getElementById('character');
  const skin = getSkin(gs.activeSkinId);
  switch (skin.id) {
    case 'cat':     el.innerHTML = buildCatHTML(); break;
    case 'boss':    el.innerHTML = buildBossHTML(); break;
    case 'teacher': el.innerHTML = buildTeacherHTML(); break;
    case 'camera':  el.innerHTML = buildCameraHTML(); break;
  }
}

function buildCatHTML() {
  return `<div id="character-inner" style="position:absolute;inset:0">
    <div class="char-body"></div>
    <div class="char-ear-l"></div><div class="char-ear-r"></div>
    <div class="char-eye-l"><div class="char-pupil-l"></div></div>
    <div class="char-eye-r"><div class="char-pupil-r"></div></div>
    <div class="char-nose"></div><div class="char-tail"></div>
  </div>`;
}

function buildBossHTML() {
  // 睡觉领导：圆脸 + 眯缝眼 + 领带
  return `<div id="character-inner" style="position:absolute;inset:0;perspective:200px">
    <div style="position:absolute;width:90px;height:90px;left:15px;top:10px;
         background:#9c27b0;border-radius:50%;"></div>
    <div style="position:absolute;top:38px;left:22px;width:22px;height:5px;
         background:#fff;border-radius:3px;"></div>
    <div style="position:absolute;top:38px;right:22px;width:22px;height:5px;
         background:#fff;border-radius:3px;"></div>
    <div style="position:absolute;top:58px;left:45px;width:30px;height:3px;
         background:#fff;border-radius:2px;transform:rotate(180deg)"></div>
    <div style="position:absolute;top:82px;left:50px;width:20px;height:28px;
         background:#7b1fa2;clip-path:polygon(50% 0%, 0% 100%, 100% 100%);"></div>
  </div>`;
}

function buildTeacherHTML() {
  // 监考老师：矩形眼镜 + 严肃表情
  return `<div id="character-inner" style="position:absolute;inset:0;perspective:200px">
    <div style="position:absolute;width:90px;height:90px;left:15px;top:10px;
         background:#43a047;border-radius:40% 40% 35% 35%;"></div>
    <div style="position:absolute;top:34px;left:18px;width:26px;height:18px;
         background:transparent;border:3px solid #fff;border-radius:4px;"></div>
    <div style="position:absolute;top:34px;right:18px;width:26px;height:18px;
         background:transparent;border:3px solid #fff;border-radius:4px;"></div>
    <div style="position:absolute;top:43px;left:44px;width:12px;height:2px;background:#fff"></div>
    <div style="position:absolute;top:60px;left:28px;width:44px;height:3px;
         background:#fff;border-radius:2px;"></div>
  </div>`;
}

function buildCameraHTML() {
  // 监控摄像头：金属盒 + 镜头
  return `<div id="character-inner" style="position:absolute;inset:0;perspective:200px">
    <div style="position:absolute;width:80px;height:60px;left:20px;top:20px;
         background:#1e88e5;border-radius:8px;border:3px solid #1565c0;"></div>
    <div style="position:absolute;width:36px;height:36px;left:42px;top:32px;
         background:#0d47a1;border-radius:50%;border:4px solid #82b1ff;"></div>
    <div style="position:absolute;width:16px;height:16px;left:52px;top:42px;
         background:#e3f2fd;border-radius:50%;"></div>
    <div style="position:absolute;top:10px;right:10px;width:10px;height:10px;
         background:#f44336;border-radius:50%;box-shadow:0 0 8px #f44336;"></div>
  </div>`;
}
```

- [ ] **步骤 2：更新 updateCharacter 以支持摄像头皮肤的扫描动效**

在 `updateCharacter` 函数的 BACK 分支中追加摄像头专属处理：

```javascript
// updateCharacter 中的 PHASE.BACK 分支替换为：
if (gs.phase === PHASE.BACK) {
  inner.style.transform = gs.activeSkinId === 'camera'
    ? 'rotateY(0deg)'   // 摄像头正面朝向，但 LED 灯变绿
    : 'scaleX(-1)';
  if (gs.activeSkinId === 'camera') {
    const led = inner.querySelector('div:last-child');
    if (led) led.style.background = '#4caf50';
  }
}
// TURNING 和 LOOKING 分支中，对 camera 的 LED 设为红色：
if (gs.activeSkinId === 'camera') {
  const led = inner && inner.querySelector('div:last-child');
  if (led) led.style.background = '#f44336';
}
```

- [ ] **步骤 3：验证 4 套皮肤的外观**

逐一选择每套皮肤后开始游戏，确认：
- 角色视觉不同
- 主题颜色跟随皮肤变化
- 转头动画正常

- [ ] **步骤 4：commit**

```bash
git add index.html
git commit -m "feat: all 4 skin character visuals"
```

---

## Task 11: Web Audio 音效

**Files:**
- Modify: `index.html` — `// ── AUDIO ──` 段

- [ ] **步骤 1：写入 Audio 模块**

```javascript
// ── AUDIO ──────────────────────────────────────────
let audioCtx = null;
let brushNode = null;
let brushGain = null;

function getAudioCtx() {
  if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  return audioCtx;
}

function startBrushSound() {
  const ctx = getAudioCtx();
  if (brushNode) return;
  brushGain = ctx.createGain();
  brushGain.gain.setValueAtTime(0, ctx.currentTime);
  brushGain.gain.linearRampToValueAtTime(0.08, ctx.currentTime + 0.05);
  brushGain.connect(ctx.destination);

  // White noise via buffer
  const bufLen = ctx.sampleRate * 2;
  const buf = ctx.createBuffer(1, bufLen, ctx.sampleRate);
  const data = buf.getChannelData(0);
  for (let i = 0; i < bufLen; i++) data[i] = Math.random() * 2 - 1;

  brushNode = ctx.createBufferSource();
  brushNode.buffer = buf;
  brushNode.loop = true;

  // Low-pass filter for a softer brush texture
  const filter = ctx.createBiquadFilter();
  filter.type = 'lowpass';
  filter.frequency.value = 2000;

  brushNode.connect(filter);
  filter.connect(brushGain);
  brushNode.start();
}

function stopBrushSound() {
  if (!brushGain) return;
  const ctx = getAudioCtx();
  brushGain.gain.linearRampToValueAtTime(0, ctx.currentTime + 0.05);
  setTimeout(() => {
    if (brushNode) { try { brushNode.stop(); } catch(e) {} brushNode = null; }
    brushGain = null;
  }, 100);
}

function playWarningSound() {
  const ctx = getAudioCtx();
  const osc = ctx.createOscillator();
  const gain = ctx.createGain();
  osc.type = 'sine';
  osc.frequency.setValueAtTime(880, ctx.currentTime);
  osc.frequency.linearRampToValueAtTime(440, ctx.currentTime + 0.15);
  gain.gain.setValueAtTime(0.15, ctx.currentTime);
  gain.gain.linearRampToValueAtTime(0, ctx.currentTime + 0.15);
  osc.connect(gain); gain.connect(ctx.destination);
  osc.start(); osc.stop(ctx.currentTime + 0.15);
}

function playDeathSound() {
  const ctx = getAudioCtx();
  const osc = ctx.createOscillator();
  const gain = ctx.createGain();
  osc.type = 'sawtooth';
  osc.frequency.setValueAtTime(200, ctx.currentTime);
  osc.frequency.linearRampToValueAtTime(50, ctx.currentTime + 0.4);
  gain.gain.setValueAtTime(0.3, ctx.currentTime);
  gain.gain.linearRampToValueAtTime(0, ctx.currentTime + 0.4);
  osc.connect(gain); gain.connect(ctx.destination);
  osc.start(); osc.stop(ctx.currentTime + 0.4);
}
```

- [ ] **步骤 2：验证音效**

- 按住刷毛区：应听到轻微白噪沙沙声
- 松手：声音应淡出
- 等角色转头：应听到短促警告音
- 被抓：应听到下行沙哑音
- 控制台无 AudioContext 报错

- [ ] **步骤 3：commit**

```bash
git add index.html
git commit -m "feat: web audio brush, warning, and death sounds"
```

---

## Task 12: 首次引导 (Tutorial)

**Files:**
- Modify: `index.html` — `// ── SCREENS ──` 段

- [ ] **步骤 1：写入 startTutorial 函数**

```javascript
function startTutorial() {
  gs.screen = STATE.TUTORIAL;
  showScreen('tutorial');

  let countdown = 3;
  const timerEl = document.getElementById('tutorial-timer');

  const interval = setInterval(() => {
    countdown--;
    if (countdown <= 0) {
      clearInterval(interval);
      finishTutorial();
    } else {
      timerEl.textContent = `${countdown} 秒后自动开始...`;
    }
  }, 1000);

  document.getElementById('btn-tutorial-skip').addEventListener('click', () => {
    clearInterval(interval);
    finishTutorial();
  }, { once: true });
}

function finishTutorial() {
  localStorage.setItem('brushmaster_tutorial_seen', '1');
  goToPlaying();
}
```

- [ ] **步骤 2：验证首次引导**

1. 清空 localStorage：`localStorage.clear()`
2. 刷新页面，点击「开始游戏」
3. 预期：显示 Tutorial 界面，倒计时从 3 开始
4. 等待 3 秒：自动进入游戏
5. 再次点击「主菜单」→「开始游戏」：应直接进入游戏，跳过 Tutorial

- [ ] **步骤 3：commit**

```bash
git add index.html
git commit -m "feat: first-time tutorial with 3s countdown"
```

---

## Task 13: Canvas 背景效果

**Files:**
- Modify: `index.html` — `// ── RENDER ──` 段（追加 Canvas 模块）

- [ ] **步骤 1：写入 Canvas 初始化和粒子系统**

```javascript
// Canvas Background
const canvas = document.getElementById('bg-canvas');
const ctx2d = canvas.getContext('2d');
let particles = [];

function resizeCanvas() {
  canvas.width = canvas.offsetWidth;
  canvas.height = canvas.offsetHeight;
}

function spawnBrushParticle() {
  if (!gs.isBrushing || gs.phase !== PHASE.BACK) return;
  const zone = document.getElementById('brush-zone');
  const rect = zone.getBoundingClientRect();
  const appRect = document.getElementById('app').getBoundingClientRect();
  particles.push({
    x: rect.left - appRect.left + Math.random() * rect.width,
    y: rect.top - appRect.top + Math.random() * 20,
    vx: (Math.random() - 0.5) * 2,
    vy: -Math.random() * 3 - 1,
    life: 1,
    size: Math.random() * 6 + 2,
  });
}

function renderCanvas() {
  ctx2d.clearRect(0, 0, canvas.width, canvas.height);
  // Spawn particles while brushing
  if (gs.isBrushing && Math.random() < 0.6) spawnBrushParticle();

  particles = particles.filter(p => p.life > 0);
  particles.forEach(p => {
    p.x += p.vx; p.y += p.vy; p.life -= 0.04;
    ctx2d.globalAlpha = p.life;
    ctx2d.fillStyle = getComputedStyle(document.documentElement).getPropertyValue('--primary').trim();
    ctx2d.beginPath();
    ctx2d.arc(p.x, p.y, p.size, 0, Math.PI * 2);
    ctx2d.fill();
  });
  ctx2d.globalAlpha = 1;
}
```

- [ ] **步骤 2：在 gameLoop 中调用 renderCanvas，在 init 中调用 resizeCanvas**

在 `gameLoop` 函数内（`requestAnimationFrame(gameLoop)` 之前）追加：
```javascript
renderCanvas();
```

在 `init()` 的第一行追加：
```javascript
resizeCanvas();
window.addEventListener('resize', resizeCanvas);
```

- [ ] **步骤 3：验证粒子效果**

开始游戏，按住刷毛区，应看到橙色粒子从底部刷毛区向上飘散。

- [ ] **步骤 4：commit**

```bash
git add index.html
git commit -m "feat: canvas brush particle effect"
```

---

## Task 14: 整体打磨 + 移动端测试

**Files:**
- Modify: `index.html`

- [ ] **步骤 1：在 `<style>` 追加连击倍率弹出动画**

```css
@keyframes comboPop {
  0%   { transform: scale(1); }
  50%  { transform: scale(1.5); }
  100% { transform: scale(1); }
}
#combo-display.pop { animation: comboPop 0.2s ease; }
```

在 `updateComboDisplay` 中触发动画：

```javascript
function updateComboDisplay() {
  const mult = BM.getMultiplier(gs.comboCount);
  const el = document.getElementById('combo-display');
  if (mult > 1) {
    el.textContent = `×${mult} 连击!`;
    el.classList.remove('pop');
    void el.offsetWidth; // 强制 reflow
    el.classList.add('pop');
  } else {
    el.textContent = '';
  }
}
```

- [ ] **步骤 2：移动端测试清单**

用 Chrome DevTools 的移动端模拟（iPhone 12 Pro，375px）逐项验证：

- [ ] 所有按钮可点击，没有溢出屏幕
- [ ] 按住刷毛区有触觉反馈（颜色变化）
- [ ] 角色居中显示，大小合适
- [ ] YOU DIED 文字不被截断
- [ ] 皮肤选择 2×2 网格正常

- [ ] **步骤 3：修复任何布局问题后 commit**

```bash
git add index.html
git commit -m "feat: combo pop animation and mobile layout fixes"
```

---

## Task 15: 验收测试清单

- [ ] **完整游玩一局（橙猫）：** 得分增加 → 连击倍率上升 → 被抓 → YOU DIED 显示分数 → 再来一局重置
- [ ] **皮肤切换：** 4 套皮肤主题色、角色外观均不同
- [ ] **监考老师假转头：** 平均 10 次转头中应有约 3 次假转头（转一半回去）
- [ ] **localStorage 持久化：** 刷新页面后最高分不丢失
- [ ] **首次引导：** 清空 localStorage 后首次进入显示 Tutorial，之后不再显示
- [ ] **×16 封顶：** 达到 12 次连击后倍率不再上升，但显示继续跳动
- [ ] **音效：** 刷毛、警告、死亡三种音效均正常

- [ ] **最终 commit**

```bash
git add index.html
git commit -m "feat: complete brush master game v1.0"
```
