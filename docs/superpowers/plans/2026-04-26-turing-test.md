# AI 还是人？实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 单文件浏览器答题游戏，展示文字片段让玩家判断 AI 或人类所写，翻牌揭晓，追连击最高记录。

**Architecture:** 单文件 `ai-or-human/index.html`，零依赖零构建。纯 HTML+CSS+JS，CSS 3D 翻牌动画，`TT` 纯逻辑命名空间，状态机 MENU→PLAYING，localStorage 持久化最高连击。

**Tech Stack:** HTML5, CSS3（transform-style: preserve-3d, @keyframes）, Vanilla JS, localStorage

---

## 文件结构

```
today-x/
└── ai-or-human/
    └── index.html   ← 全部代码，单文件
```

---

### Task 1: HTML 骨架 + CSS 变量 + 屏幕布局

**Files:**
- Create: `ai-or-human/index.html`

- [ ] **Step 1: 创建文件，写完整 HTML 骨架**

```html
<!DOCTYPE html>
<html lang="zh">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
  <title>AI 还是人？</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

    :root {
      --bg: #0f0f0f;
      --card-bg: #1e1e1e;
      --card-border: #2e2e2e;
      --text: #f0f0f0;
      --muted: #666;
      --correct: #22c55e;
      --wrong: #ef4444;
      --ai-btn: #6366f1;
      --human-btn: #f59e0b;
    }

    body {
      background: var(--bg);
      color: var(--text);
      font-family: -apple-system, BlinkMacSystemFont, 'PingFang SC', 'Hiragino Sans GB', sans-serif;
      min-height: 100dvh;
      display: flex;
      align-items: center;
      justify-content: center;
    }

    #app {
      width: 100%;
      max-width: 480px;
      min-height: 100dvh;
      position: relative;
    }

    .screen {
      position: absolute;
      inset: 0;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      padding: 32px 24px;
      gap: 20px;
    }
    .screen.hidden { display: none; }

    /* ── MENU ── */
    #menu-title {
      font-size: clamp(32px, 10vw, 52px);
      font-weight: 900;
      letter-spacing: -1px;
      text-align: center;
    }
    #menu-subtitle {
      font-size: 15px;
      color: var(--muted);
      text-align: center;
    }
    #menu-best {
      font-size: 14px;
      color: var(--muted);
    }

    /* ── BUTTONS ── */
    .btn {
      border: none; cursor: pointer; border-radius: 14px;
      font-size: 16px; font-weight: 700;
      padding: 14px 32px; transition: transform 0.1s, opacity 0.1s;
    }
    .btn:active { transform: scale(0.96); }
    .btn-primary {
      background: var(--text); color: var(--bg);
      width: 100%; max-width: 280px;
    }

    /* ── PLAYING ── */
    #hud {
      width: 100%;
      display: flex;
      justify-content: space-between;
      align-items: center;
      font-size: 14px;
      font-weight: 700;
    }
    #streak-display { color: var(--text); font-size: 18px; }
    #best-display { color: var(--muted); }

    #card-wrapper {
      width: 100%;
      perspective: 800px;
    }

    #card {
      width: 100%;
      min-height: 220px;
      position: relative;
      transform-style: preserve-3d;
      transition: transform 0.55s cubic-bezier(0.4, 0, 0.2, 1);
    }
    #card.flipped { transform: rotateY(180deg); }

    #card-front, #card-back {
      position: absolute;
      inset: 0;
      min-height: 220px;
      border-radius: 20px;
      padding: 28px 24px;
      backface-visibility: hidden;
      display: flex;
      flex-direction: column;
      justify-content: center;
    }

    #card-front {
      background: var(--card-bg);
      border: 1px solid var(--card-border);
      font-size: 17px;
      line-height: 1.75;
    }
    #card-front.correct { background: #14532d; border-color: var(--correct); }
    #card-front.wrong   { background: #450a0a; border-color: var(--wrong); }

    #card-back {
      background: var(--card-bg);
      border: 1px solid var(--card-border);
      transform: rotateY(180deg);
      gap: 14px;
    }
    #answer-label {
      font-size: 20px;
      font-weight: 900;
    }
    #hint-text {
      font-size: 13px;
      color: var(--muted);
      line-height: 1.6;
    }

    #btn-row {
      width: 100%;
      display: grid;
      grid-template-columns: 1fr 1fr;
      gap: 12px;
    }
    .answer-btn {
      border: none; cursor: pointer; border-radius: 14px;
      font-size: 15px; font-weight: 700; padding: 16px 8px;
      transition: transform 0.1s, opacity 0.1s;
      color: #fff;
    }
    .answer-btn:active { transform: scale(0.96); }
    .answer-btn:disabled { opacity: 0.4; cursor: default; transform: none; }
    #btn-ai    { background: var(--ai-btn); }
    #btn-human { background: var(--human-btn); }

    /* ── ANIMATIONS ── */
    @keyframes shake {
      0%, 100% { transform: translateX(0); }
      20%       { transform: translateX(-6px); }
      40%       { transform: translateX(6px); }
      60%       { transform: translateX(-4px); }
      80%       { transform: translateX(4px); }
    }
    #streak-display.shake { animation: shake 0.35s ease; }

    @keyframes slideInUp {
      from { opacity: 0; transform: translateY(24px); }
      to   { opacity: 1; transform: translateY(0); }
    }
    #card-wrapper.slide-in { animation: slideInUp 0.3s ease; }
  </style>
</head>
<body>
  <div id="app">

    <!-- MENU -->
    <div id="screen-menu" class="screen">
      <div id="menu-title">AI 还是人？</div>
      <div id="menu-subtitle">你能分清吗？</div>
      <div id="menu-best"></div>
      <button id="btn-start" class="btn btn-primary">开始挑战</button>
    </div>

    <!-- PLAYING -->
    <div id="screen-playing" class="screen hidden">
      <div id="hud">
        <div id="streak-display">🔥 ×<span id="streak-num">0</span></div>
        <div id="best-display">最高 <span id="best-num">0</span></div>
      </div>

      <div id="card-wrapper">
        <div id="card">
          <div id="card-front">
            <p id="question-text"></p>
          </div>
          <div id="card-back">
            <div id="answer-label"></div>
            <p id="hint-text"></p>
          </div>
        </div>
      </div>

      <div id="btn-row">
        <button id="btn-ai" class="answer-btn">🤖 AI 写的</button>
        <button id="btn-human" class="answer-btn">👤 人类写的</button>
      </div>
    </div>

  </div>
  <script>
    // JS goes here in later tasks
  </script>
</body>
</html>
```

- [ ] **Step 2: 在浏览器中打开，确认两个屏幕布局正确**

```bash
cd /home/zhcao/today-x && python3 -m http.server 8765
# 访问 http://localhost:8765/ai-or-human/
```

预期：看到黑色背景 + 主菜单标题居中显示。

- [ ] **Step 3: Commit**

```bash
git add ai-or-human/index.html
git commit -m "feat: HTML skeleton, CSS layout and card structure"
```

---

### Task 2: 题库数据（20 道题）

**Files:**
- Modify: `ai-or-human/index.html`（`<script>` 标签内）

- [ ] **Step 1: 在 `<script>` 标签内写题库数组**

将 `// JS goes here in later tasks` 替换为：

```js
// ── DATASET ────────────────────────────────────────
const QUESTIONS = [
  // ── 朋友圈 / 微博（AI）──
  {
    text: '阳光正好，微风不燥，愿每一个平凡的日子都值得被珍藏，愿你我都能在温柔里慢慢变好。',
    isAI: true,
    hint: '"微风不燥""值得被珍藏"是网络金句模板，AI 拼接痕迹明显，无具体场景'
  },
  {
    text: '今天的咖啡格外香醇，窗外的阳光透过玻璃洒在桌上，整个人都变得平静而温暖，感谢生活。',
    isAI: true,
    hint: 'AI 场景三件套：咖啡 + 阳光 + 平静，结尾"感谢生活"是固定收尾公式'
  },
  {
    text: '每一次落日都是大自然馈赠的画作，每一个清晨都是新生命的起点，生命因此而无限美丽。',
    isAI: true,
    hint: '"馈赠""无限美丽"是 AI 散文公式：自然意象 + 人生感悟 + 升华词'
  },
  // ── 朋友圈 / 微博（人类）──
  {
    text: '我妈今天做了红烧肉，我连吃了三碗饭，现在躺在沙发上完全动不了，后悔死了',
    isAI: false,
    hint: '具体数量（三碗）+ 具体后果（动不了）+ 末尾短促情绪，是人类真实流水账'
  },
  {
    text: '跟朋友吵架了，其实也没什么大事，就坐在路边哭了一会儿，然后买了杯奶茶，好多了',
    isAI: false,
    hint: '情绪有具体转折：哭 → 奶茶 → 好了，人类情感有这种具体的小解法'
  },
  {
    text: '出门忘带钥匙，打电话给我男朋友他也忘了，我们俩就在楼道里坐了一个多小时等他妈回来',
    isAI: false,
    hint: '双重失误的荒诞场景，"等他妈回来"的口语收尾，AI 不会这样写'
  },
  {
    text: '今年冬天好像特别短，还没来得及买羽绒服就开春了，也不知道算省了还是亏了',
    isAI: false,
    hint: '"算省了还是亏了"的生活小纠结，是人类才有的真实矛盾收尾'
  },
  // ── 产品评论（AI）──
  {
    text: '收到货非常满意，做工精细，质量上乘，与描述完全一致，物流速度很快，五星好评，强烈推荐！',
    isAI: true,
    hint: 'AI 评价万能模板：满意 / 精细 / 一致 / 物流 / 五星，无任何个人使用细节'
  },
  {
    text: '这款产品性价比极高，外观设计简约大方，使用体验非常舒适流畅，值得购买，会回购。',
    isAI: true,
    hint: '"性价比极高""简约大方""流畅"是 AI 评论高频词，空泛无具体感受'
  },
  // ── 产品评论（人类）──
  {
    text: '买来给我奶奶用的，她说按键有点小，但字体挺大的，凑合能用，比上一个好多了',
    isAI: false,
    hint: '有具体使用者（奶奶）、具体缺点（按键小）、比较基准，是真实反馈'
  },
  {
    text: '用了一周，充一次电能撑三天，就是充电线太短了，坐在床上够不到插座，有点烦',
    isAI: false,
    hint: '具体使用时长、具体缺点（线短够不到插座），有生活场景，是真实评价'
  },
  // ── 新闻 / 时评（AI）──
  {
    text: '在这个瞬息万变的时代，唯有不断学习与自我革新，方能在激烈的竞争中立于不败之地。',
    isAI: true,
    hint: 'AI 时评公式：时代背景 + "唯有" + "立于不败之地"，宏大空洞无具体观点'
  },
  {
    text: '科技的进步为人类带来了前所未有的便利，但同时也引发了诸多值得深思的伦理问题。',
    isAI: true,
    hint: '"但同时也"是 AI 最爱的两面都说结构，无立场，无具体事例'
  },
  {
    text: '面对当前复杂多变的国际形势，我们更需要保持清醒的头脑，坚定信念，稳步前行。',
    isAI: true,
    hint: '"复杂多变""清醒头脑""坚定信念""稳步前行"——四个套话连发'
  },
  // ── 新闻 / 时评（人类）──
  {
    text: '这个政策出来之后，我身边好几个做外贸的朋友都在问到底有没有影响，说实话现在谁也说不准',
    isAI: false,
    hint: '有具体社交圈（朋友 / 外贸）、承认不确定，是人类说话的真实语气'
  },
  {
    text: '上次出差去郑州，住的酒店 WiFi 断了三次，我问前台，前台说"在修"，修了整整三天',
    isAI: false,
    hint: '具体地点、具体次数、具体对话，是人类亲历感；AI 不会用"整整三天"来讽刺'
  },
  // ── 短诗 / 散文（AI）──
  {
    text: '你是春风，是暖阳，是我漫长冬日里最温柔的光，是我跌入低谷时的一束希望。',
    isAI: true,
    hint: 'AI 写情感惯用排比 + 意象叠加（春风 / 暖阳 / 光 / 希望），情感空洞无来由'
  },
  {
    text: '时光流逝，岁月静好，愿你我都能在这纷扰的世界里找到属于自己的一份宁静与从容。',
    isAI: true,
    hint: '"岁月静好" + "宁静与从容"是 AI 散文双重空洞，无任何具体场景支撑'
  },
  // ── 短诗 / 散文（人类）──
  {
    text: '我妈在医院排了四个小时才看上，出来的时候脸上还带着笑说没什么大事，我在外面红了眼眶',
    isAI: false,
    hint: '克制的情感张力（妈妈笑 → 我红眼眶）、具体时间，AI 会让结局更煽情'
  },
  {
    text: '下雨天坐在窗边，脑子里突然想起一个十年没联系的人，不知道他过得怎么样，也没想去问',
    isAI: false,
    hint: '"想起却不行动"是人类情感的真实矛盾，AI 会让主角去联系或感慨万千'
  },
];
```

- [ ] **Step 2: 在浏览器控制台验证题库**

打开 `http://localhost:8765/ai-or-human/`，控制台运行：

```js
console.log(QUESTIONS.length);            // 期望：20
console.log(QUESTIONS.filter(q => q.isAI).length);   // 期望：10
console.log(QUESTIONS.filter(q => !q.isAI).length);  // 期望：10
console.log(QUESTIONS.every(q => q.text && q.hint && typeof q.isAI === 'boolean')); // 期望：true
```

- [ ] **Step 3: Commit**

```bash
git add ai-or-human/index.html
git commit -m "feat: add 20-question dataset (10 AI, 10 human)"
```

---

### Task 3: 纯逻辑函数（TT 命名空间）

**Files:**
- Modify: `ai-or-human/index.html`

- [ ] **Step 1: 在题库数组后追加 TT 命名空间**

```js
// ── PURE LOGIC ─────────────────────────────────────
const TT = {};

TT.shuffle = function(arr) {
  const a = arr.slice();
  for (let i = a.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [a[i], a[j]] = [a[j], a[i]];
  }
  return a;
};

TT.isCorrect = function(guessIsAI, questionIsAI) {
  return guessIsAI === questionIsAI;
};

TT.getNextStreak = function(current, correct) {
  return correct ? current + 1 : 0;
};
```

- [ ] **Step 2: 控制台验证三个函数**

```js
// shuffle 不改变长度，不重复
const s = TT.shuffle([1,2,3,4,5]);
console.log(s.length === 5, new Set(s).size === 5); // true true

// isCorrect
console.log(TT.isCorrect(true, true));   // true
console.log(TT.isCorrect(true, false));  // false
console.log(TT.isCorrect(false, false)); // true

// getNextStreak
console.log(TT.getNextStreak(3, true));  // 4
console.log(TT.getNextStreak(3, false)); // 0
console.log(TT.getNextStreak(0, true));  // 1
```

- [ ] **Step 3: Commit**

```bash
git add ai-or-human/index.html
git commit -m "feat: TT pure logic namespace (shuffle, isCorrect, getNextStreak)"
```

---

### Task 4: 游戏状态对象 + 状态机

**Files:**
- Modify: `ai-or-human/index.html`

- [ ] **Step 1: 追加状态常量和游戏状态对象**

```js
// ── STATE ──────────────────────────────────────────
const STATE = { MENU: 'MENU', PLAYING: 'PLAYING' };

const gs = {
  screen: STATE.MENU,
  deck: [],          // 洗好的题库副本
  deckIndex: 0,      // 当前题目指针
  streak: 0,
  bestStreak: 0,
  answered: false,   // 当前题是否已作答（防止重复点击）
};

function showScreen(name) {
  document.getElementById('screen-menu').classList.toggle('hidden', name !== 'menu');
  document.getElementById('screen-playing').classList.toggle('hidden', name !== 'playing');
}

function goToMenu() {
  gs.screen = STATE.MENU;
  document.getElementById('menu-best').textContent =
    gs.bestStreak > 0 ? `历史最高连击 ${gs.bestStreak}` : '';
  showScreen('menu');
}

function goToPlaying() {
  gs.screen = STATE.PLAYING;
  gs.streak = 0;
  gs.deck = TT.shuffle(QUESTIONS);
  gs.deckIndex = 0;
  gs.answered = false;
  showScreen('playing');
  loadQuestion();
}

function loadQuestion() {
  if (gs.deckIndex >= gs.deck.length) {
    gs.deck = TT.shuffle(QUESTIONS);
    gs.deckIndex = 0;
  }
  const q = gs.deck[gs.deckIndex];
  document.getElementById('question-text').textContent = q.text;
  const cardFront = document.getElementById('card-front');
  cardFront.classList.remove('correct', 'wrong');
  document.getElementById('card').classList.remove('flipped');
  document.getElementById('btn-ai').disabled = false;
  document.getElementById('btn-human').disabled = false;
  gs.answered = false;
}
```

- [ ] **Step 2: 控制台验证**

```js
goToPlaying();
console.log(gs.screen);        // "PLAYING"
console.log(gs.deck.length);   // 20
console.log(typeof gs.deck[0].text); // "string"
```

- [ ] **Step 3: Commit**

```bash
git add ai-or-human/index.html
git commit -m "feat: game state object, state machine, loadQuestion"
```

---

### Task 5: 答题交互 + 翻牌揭晓

**Files:**
- Modify: `ai-or-human/index.html`

- [ ] **Step 1: 追加答题逻辑**

```js
// ── ANSWER ─────────────────────────────────────────
function submitAnswer(guessIsAI) {
  if (gs.answered) return;
  gs.answered = true;

  const q = gs.deck[gs.deckIndex];
  const correct = TT.isCorrect(guessIsAI, q.isAI);

  // 正面变色
  const cardFront = document.getElementById('card-front');
  cardFront.classList.add(correct ? 'correct' : 'wrong');

  // 填充背面内容
  document.getElementById('answer-label').textContent =
    q.isAI ? '🤖 AI 写的' : '👤 人类写的';
  document.getElementById('hint-text').textContent = q.hint;

  // 禁用按钮
  document.getElementById('btn-ai').disabled = true;
  document.getElementById('btn-human').disabled = true;

  // 翻牌
  document.getElementById('card').classList.add('flipped');

  // 更新连击
  const newStreak = TT.getNextStreak(gs.streak, correct);
  gs.streak = newStreak;
  if (gs.streak > gs.bestStreak) {
    gs.bestStreak = gs.streak;
    localStorage.setItem('tt_best_streak', gs.bestStreak);
  }
  updateHUD(correct);

  // 1.5 秒后进入下一题
  setTimeout(() => {
    gs.deckIndex++;
    advanceCard();
  }, 1500);
}

function advanceCard() {
  const wrapper = document.getElementById('card-wrapper');
  wrapper.classList.remove('slide-in');
  // 强制 reflow
  void wrapper.offsetWidth;
  wrapper.classList.add('slide-in');
  loadQuestion();
}
```

- [ ] **Step 2: 临时测试（控制台）**

```js
goToPlaying();
submitAnswer(true);  // 第一题若是 AI 则绿色翻牌，否则红色
```

预期：卡片颜色变化，0.55s 后翻转，背面显示答案和 hint。

- [ ] **Step 3: Commit**

```bash
git add ai-or-human/index.html
git commit -m "feat: answer submission, card flip reveal, auto-advance"
```

---

### Task 6: 连击 HUD + 抖动动画

**Files:**
- Modify: `ai-or-human/index.html`

- [ ] **Step 1: 追加 HUD 更新函数**

```js
// ── HUD ────────────────────────────────────────────
function updateHUD(correct) {
  const streakEl = document.getElementById('streak-display');
  document.getElementById('streak-num').textContent = gs.streak;
  document.getElementById('best-num').textContent = gs.bestStreak;

  if (!correct) {
    streakEl.classList.remove('shake');
    void streakEl.offsetWidth; // 强制 reflow 让动画重新触发
    streakEl.classList.add('shake');
    streakEl.addEventListener('animationend', () => {
      streakEl.classList.remove('shake');
    }, { once: true });
  }
}
```

- [ ] **Step 2: 验证抖动动画**

控制台运行：
```js
goToPlaying();
gs.streak = 5;
updateHUD(false);  // 连击归零，数字应抖动
```

预期：左上角连击数字抖动一次，值变为 0。

- [ ] **Step 3: Commit**

```bash
git add ai-or-human/index.html
git commit -m "feat: streak HUD with shake animation on reset"
```

---

### Task 7: Input 事件绑定 + localStorage + 主菜单

**Files:**
- Modify: `ai-or-human/index.html`

- [ ] **Step 1: 追加初始化函数**

```js
// ── INIT ───────────────────────────────────────────
function init() {
  // 加载最高记录
  gs.bestStreak = parseInt(localStorage.getItem('tt_best_streak') || '0', 10);

  // 绑定按钮
  document.getElementById('btn-start').addEventListener('click', goToPlaying);
  document.getElementById('btn-ai').addEventListener('click', () => submitAnswer(true));
  document.getElementById('btn-human').addEventListener('click', () => submitAnswer(false));

  // 初始画面
  goToMenu();
}

init();
```

- [ ] **Step 2: 完整游玩测试**

```bash
# 浏览器访问 http://localhost:8765/ai-or-human/
```

逐项验证：
1. 主菜单显示正常
2. 点击「开始挑战」进入游戏
3. 点击「AI 写的」或「人类写的」触发翻牌
4. 答对卡片变绿，答错变红，背面显示 hint
5. 1.5 秒后自动进入下一题
6. 连击数正确更新
7. 答错时连击抖动归零
8. 刷新页面后最高连击保留

- [ ] **Step 3: Commit**

```bash
git add ai-or-human/index.html
git commit -m "feat: input binding, localStorage persistence, menu wiring"
```

---

### Task 8: 主菜单返回 + 移动端适配

**Files:**
- Modify: `ai-or-human/index.html`

- [ ] **Step 1: 在 HUD 区域加"返回菜单"入口，并修复移动端样式**

在 CSS 中追加（`@keyframes slideInUp` 后面）：

```css
#back-btn {
  position: absolute;
  bottom: 28px;
  left: 50%;
  transform: translateX(-50%);
  background: transparent;
  border: none;
  color: var(--muted);
  font-size: 13px;
  cursor: pointer;
  padding: 8px 16px;
}
#back-btn:hover { color: var(--text); }

@media (max-height: 600px) {
  #card-front, #card-back { min-height: 160px; padding: 18px 16px; }
  #card-front { font-size: 15px; }
}
```

在 `#screen-playing` div 内（`#btn-row` 之后）追加：

```html
<button id="back-btn">返回菜单</button>
```

在 `init()` 的绑定区域追加：

```js
document.getElementById('back-btn').addEventListener('click', goToMenu);
```

- [ ] **Step 2: 测试移动端尺寸**

Chrome DevTools → iPhone 12 Pro（390×844），验证：
- 卡片文字可读，不被截断
- 两个按钮可点击，触摸区域足够大
- 返回菜单按钮可见

- [ ] **Step 3: 最终 Commit**

```bash
git add ai-or-human/index.html
git commit -m "feat: back button, mobile layout polish — v1.0"
```

---

### Task 9: 接入首页导航

**Files:**
- Modify: `today-x/index.html`

- [ ] **Step 1: 在根目录 `index.html` 的 `.grid` 中追加新卡片**

在 `brush-master` 的 `<a class="card">` 之后追加：

```html
<a class="card" href="ai-or-human/">
  <div class="card-emoji">🤖</div>
  <div class="card-title">AI 还是人？</div>
  <div class="card-desc">Turing Test 答题游戏 — 判断这段文字是 AI 写的还是人类写的，追最高连击。</div>
  <div class="card-tag">游戏</div>
  <div class="card-date">2026-04-26</div>
</a>
```

- [ ] **Step 2: 验证首页**

```bash
# 访问 http://localhost:8765/
```

预期：两张卡片并列，点击「AI 还是人？」跳转到游戏。

- [ ] **Step 3: Commit 并推送**

```bash
git add today-x/index.html
git commit -m "feat: add AI-or-Human to homepage gallery"
git push
```
