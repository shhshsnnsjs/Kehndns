<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
  <title>Mines: Diamonds Edition</title>
  <script src="https://telegram.org/js/telegram-web-app.js"></script>
  <style>
    :root {
      --bg: #0f0f13;
      --cell: #1a1a22;
      --accent: #00ff9d;
      --danger: #ff4757;
      --text: #ffffff;
      --gold: #f1c40f;
    }
    body {
      background: var(--bg);
      color: var(--text);
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
      margin: 0;
      padding: 0;
      overflow: hidden;
      display: flex;
      flex-direction: column;
      height: 100vh;
    }
    header {
      padding: 10px 15px;
      display: flex;
      justify-content: space-between;
      align-items: center;
      border-bottom: 1px solid #2a2a35;
    }
    .stat-box {
      display: flex;
      gap: 12px;
    }
    .stat {
      display: flex;
      align-items: center;
      gap: 6px;
      font-weight: 600;
    }
    .star {
      color: var(--gold);
    }
    .grid {
      flex: 1;
      display: grid;
      grid-template-columns: repeat(5, 1fr);
      gap: 6px;
      padding: 12px;
      overflow: auto;
    }
    .cell {
      aspect-ratio: 1;
      background: var(--cell);
      border-radius: 8px;
      display: flex;
      align-items: center;
      justify-content: center;
      cursor: pointer;
      user-select: none;
    }
    .cell.mine {
      background: var(--danger);
      color: white;
    }
    .cell.safe {
      background: #2d2d3a;
      color: var(--accent);
    }
    .controls {
      padding: 12px;
      background: var(--bg);
      border-top: 1px solid #2a2a35;
    }
    .bet-row {
      display: flex;
      align-items: center;
      gap: 10px;
      margin-bottom: 10px;
    }
    .bet-input {
      flex: 1;
      background: #2d2d3a;
      border: none;
      color: white;
      padding: 10px;
      border-radius: 8px;
      text-align: right;
      font-size: 16px;
    }
    .btn-inc-dec {
      width: 40px;
      height: 40px;
      border-radius: 8px;
      border: none;
      background: #2d2d3a;
      color: white;
      font-weight: bold;
      cursor: pointer;
    }
    .btn-bet {
      width: 100%;
      background: var(--accent);
      color: #000;
      border: none;
      padding: 14px;
      border-radius: 12px;
      font-weight: bold;
      font-size: 16px;
      cursor: pointer;
    }
    .multipliers {
      display: flex;
      gap: 8px;
      padding: 0 12px 12px;
      flex-wrap: wrap;
    }
    .mult-badge {
      background: #2d2d3a;
      padding: 6px 10px;
      border-radius: 20px;
      font-size: 13px;
    }
    .autobet-row {
      display: flex;
      align-items: center;
      gap: 10px;
      padding: 0 12px;
    }
    .toggle {
      width: 40px;
      height: 22px;
      border-radius: 11px;
      background: #2d2d3a;
      position: relative;
      cursor: pointer;
    }
    .toggle::after {
      content: '';
      position: absolute;
      top: 2px;
      left: 2px;
      width: 18px;
      height: 18px;
      background: white;
      border-radius: 50%;
      transition: transform 0.2s;
    }
    .toggle.active::after {
      transform: translateX(18px);
      background: var(--accent);
    }
    .mines-count {
      display: flex;
      gap: 8px;
      padding: 0 12px 12px;
      flex-wrap: wrap;
    }
    .mine-option {
      background: #2d2d3a;
      padding: 6px 10px;
      border-radius: 20px;
      font-size: 13px;
      cursor: pointer;
    }
    .mine-option.selected {
      background: var(--accent);
      color: #000;
    }
  </style>
</head>
<body>
  <header>
    <span>Назад</span>
    <div class="stat-box">
      <div class="stat"><span class="star">★</span> <span id="balance">1000</span></div>
      <div class="stat"><span>3</span></div>
    </div>
  </header>

  <div class="grid" id="grid"></div>

  <div class="controls">
    <div class="bet-row">
      <input type="text" class="bet-input" id="betInput" value="25" readonly />
      <button class="btn-inc-dec" onclick="changeBet(-10)">−</button>
      <button class="btn-inc-dec" onclick="changeBet(10)">+</button>
    </div>
    <button class="btn-bet" id="betBtn" onclick="placeBet()">Bet ★</button>

    <div class="autobet-row">
      <label class="toggle" id="autobetToggle" onclick="toggleAutobet()"></label>
      <span>Autobet</span>
      <div style="margin-left: auto; display: flex; gap: 6px;">
        <div class="mine-option">5</div>
        <div class="mine-option">10</div>
        <div class="mine-option">∞</div>
      </div>
    </div>

    <div class="mines-count">
      <div class="mine-option selected">2</div>
      <div class="mine-option">3</div>
      <div class="mine-option">5</div>
      <div class="mine-option">7</div>
    </div>

    <div class="multipliers">
      <span class="mult-badge">×1.07</span>
      <span class="mult-badge">×1.23</span>
      <span class="mult-badge">×1.41</span>
      <span class="mult-badge">×1.64</span>
      <span class="mult-badge">×1.91</span>
    </div>
  </div>

  <script>
    const tg = window.Telegram.WebApp;
    tg.ready();
    tg.expand();

    const gridEl = document.getElementById('grid');
    const balanceEl = document.getElementById('balance');
    const betInput = document.getElementById('betInput');
    const betBtn = document.getElementById('betBtn');
    const autobetToggle = document.getElementById('autobetToggle');

    let balance = 1000;
    let bet = 25;
    let minesCount = 3;
    let autobet = false;
    let gameActive = false;

    function initGrid() {
      gridEl.innerHTML = '';
      const cells = 25;
      const mineIndices = new Set();
      while (mineIndices.size < minesCount) {
        mineIndices.add(Math.floor(Math.random() * cells));
      }
      for (let i = 0; i < cells; i++) {
        const cell = document.createElement('div');
        cell.className = 'cell';
        cell.dataset.index = i;
        cell.dataset.isMine = mineIndices.has(i) ? 'true' : 'false';
        cell.innerHTML = '♡';
        cell.onclick = () => handleCellClick(cell);
        gridEl.appendChild(cell);
      }
    }

    function handleCellClick(cell) {
      if (!gameActive || cell.classList.contains('safe') || cell.classList.contains('mine')) return;

      if (cell.dataset.isMine === 'true') {
        cell.classList.add('mine');
        cell.innerHTML = '💣';
        gameActive = false;
        balanceEl.innerText = balance;
        betBtn.disabled = false;
        betBtn.innerText = 'Bet ★';
      } else {
        cell.classList.add('safe');
        const multiplier = 1 + (Math.random() * 0.9); // простая логика множителя
        balance += Math.floor(bet * multiplier);
        balanceEl.innerText = balance;
        if (autobet) placeBet();
      }
    }

    function placeBet() {
      if (!gameActive) {
        if (balance < bet) {
          alert('Недостаточно алмазов!');
          return;
        }
        balance -= bet;
        balanceEl.innerText = balance;
        gameActive = true;
        betBtn.disabled = true;
        betBtn.innerText = 'Playing...';
        initGrid();
      }
    }

    function changeBet(delta) {
      let val = parseInt(betInput.value) || 0;
      val = Math.max(1, val + delta);
      betInput.value = val;
      bet = val;
    }

    function toggleAutobet() {
      autobet = !autobet;
      autobetToggle.classList.toggle('active', autobet);
    }

    // Инициализация
    initGrid();
  </script>
</body>
</html>
