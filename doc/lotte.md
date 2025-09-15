<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>가위바위보 - 프롬프트 1</title>
  <style>
    :root {
      --bg: #0f172a; --panel: #0b1020; --muted: #94a3b8; --text: #e5e7eb;
      --border: #1f2937; --accent: #22d3ee; --good: #34d399;
    }
    * { box-sizing: border-box; }
    html, body { height: 100%; }
    body {
      margin: 0; font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial;
      color: var(--text); background: radial-gradient(1200px 600px at 20% -10%, #0b1225 0%, var(--bg) 60%), #0f172a;
      display: grid; grid-template-rows: auto 1fr auto; gap: 16px; padding: 16px;
    }
    h1 { margin: 0; font-size: 20px; }
    .section { background: var(--panel); border: 1px solid var(--border); border-radius: 12px; padding: 16px; }
    .top { display: grid; gap: 12px; }
    .row { display: flex; gap: 10px; align-items: center; flex-wrap: wrap; }
    .row label { color: var(--muted); font-size: 12px; }
    input[type="number"] { width: 80px; padding: 6px 8px; border: 1px solid var(--border); border-radius: 8px; background: #0a0f1e; color: var(--text); }
    .grid { display: grid; gap: 10px; }
    @media (min-width: 720px) { .grid { grid-template-columns: repeat(5, 1fr); } }
    .participant { display: grid; gap: 8px; padding: 10px; border: 1px solid var(--border); border-radius: 10px; background: rgba(255,255,255,0.02); }
    .participant label { font-size: 12px; color: var(--muted); }
    input[type="text"], select {
      width: 100%; padding: 8px 10px; border-radius: 8px; border: 1px solid var(--border); background: #0a0f1e; color: var(--text);
    }
    .score { display: inline-flex; gap: 6px; align-items: center; padding: 4px 8px; border-radius: 999px;
      background: rgba(52, 211, 153, 0.12); border: 1px solid rgba(52, 211, 153, 0.25); color: #a7f3d0; font-size: 12px; width: max-content; }
    .middle { display: grid; justify-items: center; text-align: center; padding: 24px; }
    .emoji { font-size: 80px; line-height: 1; filter: drop-shadow(0 6px 18px rgba(0,0,0,0.35)); user-select: none; }
    .caption { margin-top: 8px; font-size: 13px; color: var(--muted); }
    .rolling { animation: roll 420ms linear infinite; }
    @keyframes roll { from { transform: translateY(0); } to { transform: translateY(-4px); } }
    .bottom { display: flex; gap: 12px; align-items: center; justify-content: center; flex-wrap: wrap; }
    button {
      appearance: none; border: 1px solid rgba(59,130,246,0.4); background: linear-gradient(180deg, #3b82f6, #2563eb);
      color: #fff; padding: 10px 16px; border-radius: 10px; font-weight: 600; cursor: pointer;
    }
    button:disabled { opacity: .6; cursor: not-allowed; }
    .timer { min-width: 64px; text-align: center; padding: 8px 12px; border: 1px dashed var(--border); border-radius: 10px; color: var(--accent); font-variant-numeric: tabular-nums; font-weight: 700; }
    .legend { color: var(--muted); font-size: 12px; text-align: center; }
  </style>
</head>
<body>
  <div class="section top">
    <h1>프롬프트 1: 참가자 설정</h1>
    <div class="row">
      <label for="count">참가자 수 (1~5)</label>
      <input id="count" type="number" min="1" max="5" value="3" />
    </div>
    <div class="legend">각 참가자는 자신의 선택으로 컴퓨터를 이기면 상단의 "이긴 횟수"가 +1 됩니다. (가위✌, 바위✊, 보✋)</div>
    <div class="grid" id="participants"></div>
  </div>

  <div class="section middle">
    <div class="emoji" id="computer-emoji" aria-live="polite">❔</div>
    <div class="caption" id="computer-text">대기 중…</div>
  </div>

  <div class="section bottom">
    <button id="start-btn">시작</button>
    <div class="timer" id="timer">5.0s</div>
  </div>

  <script>
    const choices = [
      { key: 'scissors', label: '가위', emoji: '✌' },
      { key: 'rock',     label: '바위', emoji: '✊' },
      { key: 'paper',    label: '보',   emoji: '✋' },
    ];
    const winsAgainst = { rock: 'scissors', scissors: 'paper', paper: 'rock' };
    const randChoice = () => choices[Math.floor(Math.random() * choices.length)];
    const el = id => document.getElementById(id);

    const participantsWrap = el('participants');
    const countInput = el('count');
    const startBtn = el('start-btn');
    const timerEl = el('timer');
    const compEmoji = el('computer-emoji');
    const compText = el('computer-text');

    let scores = [];
    let countdownInterval = null;
    let rollingTimeout = null;
    let rollingAnimOn = false;

    function participantTemplate(i) {
      const wrap = document.createElement('div');
      wrap.className = 'participant';
      wrap.innerHTML = `
        <div>
          <label for="name-${i}">이름 (${i+1})</label>
          <input id="name-${i}" type="text" placeholder="이름 입력" />
        </div>
        <div>
          <label for="pick-${i}">선택</label>
          <select id="pick-${i}">
            <option value="scissors">✌ 가위</option>
            <option value="rock">✊ 바위</option>
            <option value="paper">✋ 보</option>
          </select>
        </div>
        <div class="score"><span>이긴 횟수</span><strong id="score-${i}">0</strong></div>
      `;
      return wrap;
    }

    function renderParticipants(n) {
      participantsWrap.innerHTML = '';
      scores = Array.from({ length: n }, () => 0);
      for (let i = 0; i < n; i++) {
        participantsWrap.appendChild(participantTemplate(i));
      }
    }

    function setComputerDisplay(choiceObj) {
      if (!choiceObj) {
        compEmoji.textContent = '❔';
        compText.textContent = '대기 중…';
        return;
      }
      compEmoji.textContent = choiceObj.emoji;
      compText.textContent = choiceObj.label;
    }

    function setDisabled(disabled) {
      startBtn.disabled = disabled;
      const n = Number(countInput.value);
      countInput.disabled = disabled;
      for (let i = 0; i < n; i++) {
        el(`name-${i}`).disabled = disabled;
        el(`pick-${i}`).disabled = disabled;
      }
    }

    function startRolling() {
      rollingAnimOn = true;
      compEmoji.classList.add('rolling');
      let i = 0;
      const interval = setInterval(() => {
        if (!rollingAnimOn) return clearInterval(interval);
        compEmoji.textContent = choices[i % 3].emoji;
        i++;
      }, 120);
      return interval;
    }

    function stopRolling(rollingInterval) {
      rollingAnimOn = false;
      compEmoji.classList.remove('rolling');
      clearInterval(rollingInterval);
    }

    function updateScoresByComputer(computerKey) {
      const n = Number(countInput.value);
      for (let i = 0; i < n; i++) {
        const chosen = el(`pick-${i}`).value;
        if (winsAgainst[chosen] === computerKey) {
          scores[i] += 1;
          el(`score-${i}`).textContent = String(scores[i]);
        }
      }
    }

    function formatSec(sec) { return `${sec.toFixed(1)}s`; }

    function startRound() {
      setDisabled(true);
      setComputerDisplay(null);

      const rollingInterval = startRolling();

      let remain = 5.0;
      timerEl.textContent = formatSec(remain);
      countdownInterval = setInterval(() => {
        remain = Math.max(0, remain - 0.1);
        timerEl.textContent = formatSec(remain);
      }, 100);

      rollingTimeout = setTimeout(() => {
        clearInterval(countdownInterval);
        timerEl.textContent = '0.0s';
        stopRolling(rollingInterval);

        const comp = randChoice();
        setComputerDisplay(comp);

        updateScoresByComputer(comp.key);
        setDisabled(false);
      }, 5000);
    }

    // Events
    countInput.addEventListener('input', () => {
      let n = Math.max(1, Math.min(5, Number(countInput.value) || 1));
      countInput.value = String(n);
      renderParticipants(n);
    });

    startBtn.addEventListener('click', () => {
      if (rollingTimeout) clearTimeout(rollingTimeout);
      if (countdownInterval) clearInterval(countdownInterval);
      startRound();
    });

    // Init
    renderParticipants(Number(countInput.value));
    setComputerDisplay(null);
    timerEl.textContent = formatSec(5.0);
  </script>
</body>
</html>