<script lang="ts">
  import { onMount, onDestroy, createEventDispatcher } from 'svelte';
  const dispatch = createEventDispatcher();

  // Grid size
  const ROWS = 5, COLS = 10;

  // Session timing (3 minutes)
  const SESSION_MS = 180_000;

  // Storage keys
  const THEME_KEY = 'upmrc_theme';
  const LOG_KEY = 'dot_top5';

  // Options
  let targetCount = 1; // number of 4-dot cells per board (1..3)

  // UI state
  let showHelp = true;      // popup visibility
  let showSummary = false;  // show summary panel in popup after End
  let timeLeft = SESSION_MS;
  let timerId: number | null = null;
  let sessionStartedAt = 0;
  let boardsCleared = 0;

  // Records
  type RecordItem = { when: number; durationMs: number; boards: number; targetsPerBoard: number };
  let top5: RecordItem[] = loadLog();

  // Theme
  let theme: 'light' | 'dark' = 'light';

  // Board state
  type Cell = {
    dots: number;
    pts: { x: number; y: number }[]; // scattered points in [0,1]x[0,1] (we constrain to 0.2..0.8)
    isTarget: boolean;
    cleared: boolean;
  };
  let board: Cell[][] = [];
  let targetsRemaining = 0;

  // Cursor (spawn bottom row, middle col; facing North)
  type Facing = 'N' | 'E' | 'S' | 'W';
  let curR = ROWS - 1;
  let curC = Math.floor(COLS / 2);
  let face: Facing = 'N';

  // ---------- THEME ----------
  function applyTheme(next: 'light' | 'dark') {
    theme = next;
    document.documentElement.setAttribute('data-theme', next);
    localStorage.setItem(THEME_KEY, next);
  }
  function toggleTheme() { applyTheme(theme === 'light' ? 'dark' : 'light'); }

  // ---------- INPUT ----------
  // Right hand: I (forward/up), M (back/down), J (turn left), L (turn right)
  // Left hand:  W (confirm/cancel when on 4-dot cell)
  function onKeyDown(e: KeyboardEvent) {
    const k = e.key.toUpperCase();
    if (showHelp) return;

    if (k === 'J') { rotateLeft(); return; }
    if (k === 'L') { rotateRight(); return; }
    if (k === 'I') { stepForward(); return; }
    if (k === 'M') { stepBackward(); return; }
    if (k === 'W') { confirmCell(); return; }
  }

  function rotateLeft() {
    face = face === 'N' ? 'W' : face === 'W' ? 'S' : face === 'S' ? 'E' : 'N';
  }
  function rotateRight() {
    face = face === 'N' ? 'E' : face === 'E' ? 'S' : face === 'S' ? 'W' : 'N';
  }
  function stepForward() {
    if (face === 'N') curR = Math.max(0, curR - 1);
    if (face === 'S') curR = Math.min(ROWS - 1, curR + 1);
    if (face === 'E') curC = Math.min(COLS - 1, curC + 1);
    if (face === 'W') curC = Math.max(0, curC - 1);
  }
  function stepBackward() {
    if (face === 'N') curR = Math.min(ROWS - 1, curR + 1);
    if (face === 'S') curR = Math.max(0, curR - 1);
    if (face === 'E') curC = Math.max(0, curC - 1);
    if (face === 'W') curC = Math.min(COLS - 1, curC + 1);
  }

  function confirmCell() {
    const cell = board[curR][curC];
    if (cell.isTarget && !cell.cleared) {
      cell.cleared = true;
      targetsRemaining--;
      if (targetsRemaining <= 0) {
        boardsCleared++;
        newBoard();
      }
    }
  }

  // ---------- BOARD ----------
  function newBoard() {
    // distribution for non-4-dot cells: 2→15%, 3→25%, 5→35%, 6→25%
    function drawCount(): number {
      const r = Math.random();
      if (r < 0.15) return 2;
      if (r < 0.40) return 3;
      if (r < 0.75) return 5;
      return 6;
    }

    // choose target positions
    const totalCells = ROWS * COLS;
    const targetPositions = new Set<number>();
    while (targetPositions.size < targetCount) {
      targetPositions.add(Math.floor(Math.random() * totalCells));
    }

    const next: Cell[][] = [];
    for (let r = 0; r < ROWS; r++) {
      const row: Cell[] = [];
      for (let c = 0; c < COLS; c++) {
        const idx = r * COLS + c;
        const isTarget = targetPositions.has(idx);
        const dots = isTarget ? 4 : drawCount();
        row.push({
          dots,
          isTarget,
          cleared: false,
          pts: scatterDotsNonOverlapping(dots)
        });
      }
      next.push(row);
    }
    board = next;
    targetsRemaining = targetCount;

    // reset cursor
    curR = ROWS - 1;
    curC = Math.floor(COLS / 2);
    face = 'N';
  }

  // dots in inner 60% (0.2..0.8), non-overlapping
  function scatterDotsNonOverlapping(n: number) {
    const pts: { x: number; y: number }[] = [];
    const PAD = 0.20;      // 20% blank edge → 60% inner area
    const MIN = PAD, MAX = 1 - PAD;
    const MIN_DIST = 0.12; // separation between dot centers (relative to cell)
    const MAX_TRIES = 200;

    function ok(nx: number, ny: number) {
      for (const p of pts) {
        const dx = nx - p.x, dy = ny - p.y;
        if (dx * dx + dy * dy < MIN_DIST * MIN_DIST) return false;
      }
      return true;
    }
    let placed = 0, tries = 0;
    while (placed < n && tries < MAX_TRIES) {
      const x = MIN + Math.random() * (MAX - MIN);
      const y = MIN + Math.random() * (MAX - MIN);
      if (ok(x, y)) { pts.push({ x, y }); placed++; }
      tries++;
    }
    return pts;
  }

  // ---------- SESSION ----------
  function startSession() {
    showHelp = false; showSummary = false;
    boardsCleared = 0;
    timeLeft = SESSION_MS;
    sessionStartedAt = Date.now();
    newBoard();
    if (timerId) clearInterval(timerId);
    timerId = window.setInterval(() => {
      timeLeft = Math.max(0, timeLeft - 1000);
      if (timeLeft <= 0) endSession();
    }, 1000) as unknown as number;
  }

  function endSession() {
    if (timerId) { clearInterval(timerId); timerId = null; }
    // Save record
    const rec: RecordItem = {
      when: sessionStartedAt || Date.now(),
      durationMs: (sessionStartedAt ? Date.now() - sessionStartedAt : 0),
      boards: boardsCleared,
      targetsPerBoard: targetCount
    };
    persistRecord(rec);
    top5 = loadLog();
    showHelp = true;
    showSummary = true; // open popup with summary section visible
  }

  // Back to home: ALWAYS clear timers + normalize state, then emit
  function backToHome() {
    if (timerId) { clearInterval(timerId); timerId = null; }
    showHelp = true;
    showSummary = false;
    timeLeft = SESSION_MS;
    boardsCleared = 0;
    dispatch('back');
  }

  // ---------- LIFECYCLE ----------
  let wrapEl: HTMLElement | null = null;  // the area that holds the board
  let boardEl: HTMLElement | null = null; // the actual board grid
  let ro: ResizeObserver | null = null;

  function layoutBoard() {
    if (!wrapEl || !boardEl) return;
    const cw = wrapEl.clientWidth;     // available width inside the 5% inset and below the topbar
    const ch = wrapEl.clientHeight;    // available height (everything below topbar)
    // Board aspect = 2:1 (width:height). Fit inside cw × ch:
    // width cannot exceed cw, and height = width/2 cannot exceed ch → width <= 2*ch
    const width = Math.floor(Math.min(cw, 2 * ch));
    const height = Math.floor(width / 2);
    boardEl.style.width = width + 'px';
    boardEl.style.height = height + 'px';
  }

  onMount(() => {
    window.addEventListener('keydown', onKeyDown);
    const saved = (localStorage.getItem(THEME_KEY) as 'light' | 'dark' | null) ?? 'light';
    applyTheme(saved);

    ro = new ResizeObserver(layoutBoard);
    if (wrapEl) ro.observe(wrapEl);
    // initial
    setTimeout(layoutBoard, 0);
  });

  onDestroy(() => {
    window.removeEventListener('keydown', onKeyDown);
    if (timerId) clearInterval(timerId);
    if (ro && wrapEl) ro.unobserve(wrapEl);
  });

  // ---------- LOGGING ----------
  function loadLog(): RecordItem[] {
    try {
      const raw = localStorage.getItem(LOG_KEY);
      const arr = raw ? JSON.parse(raw) : [];
      return Array.isArray(arr) ? arr : [];
    } catch { return []; }
  }
  function persistRecord(r: RecordItem) {
    const arr = loadLog();
    arr.unshift(r);
    while (arr.length > 5) arr.pop();
    localStorage.setItem(LOG_KEY, JSON.stringify(arr));
  }
  function clearLog() {
    if (!confirm('Clear saved Dot Test records?')) return;
    localStorage.removeItem(LOG_KEY);
    top5 = [];
  }

  // ---------- MISC ----------
  function mmss(ms: number) {
    const s = Math.ceil(ms / 1000);
    const m = Math.floor(s / 60);
    const sec = s % 60;
    return `${m}:${sec.toString().padStart(2, '0')}`;
  }
</script>

<style>
  :root { --bg:#0b0b0e; --panel:#15161a; --text:#e8eaed; --muted:#8a8f98; --border:#2a2d34; --accent:#1d4ed8; --cell:#ffffff; --dot:#111; }
  :root[data-theme='light'] { --bg:#f7f8fb; --panel:#ffffff; --text:#0f172a; --muted:#5b6270; --border:#e5e7eb; --accent:#1d4ed8; --cell:#ffffff; --dot:#111; }

  /* Fill screen, keep 5% margins on all sides */
  .stage { position:fixed; inset:5vh 5vw; display:flex; flex-direction:column; gap:10px; }

  .topbar {
    display:flex; align-items:center; justify-content:space-between; gap:12px;
    background:var(--panel); border:1px solid var(--border);
    border-radius:12px; padding:10px 14px;
  }
  .title { font-weight:700; }
  .muted { color:var(--muted); }
  .btn { background:var(--panel); color:var(--text); border:1px solid var(--border);
    padding:8px 12px; border-radius:10px; cursor:pointer; }
  .btn.primary { background:#1d4ed8; color:#fff; border:none; }

  .board-wrap {
    /* This is the area the board must fit inside (below topbar, within 5% insets) */
    flex:1; display:grid; place-items:center; min-height:0;
  }

  /* The board will get its width/height set inline by JS so it never overflows */
  .board {
    display:grid;
    grid-template-rows:repeat(5,1fr);
    grid-template-columns:repeat(10,1fr);
    gap:0; background:var(--bg); border-radius:12px;
  }

  .cell {
    position:relative; background:var(--cell);
    box-shadow: inset 0 0 0 1px var(--border);
  }

  /* Inner 60% area for dots (outer acts as blank border) */
  .inner { position:absolute; left:20%; top:20%; width:60%; height:60%; pointer-events:none; }

  .dot {
    position:absolute; width:8px; height:8px; border-radius:50%;
    background:var(--dot); transform:translate(-50%,-50%);
  }

  /* Cursor frame */
  .cursor { position:absolute; inset:0; outline:2px solid var(--accent); outline-offset:-2px; border-radius:2px; pointer-events:none; }
  .cursor .arrow { position:absolute; width:0; height:0; border-style:solid; }

  /* Arrow at the BOTTOM pointing UP when facing North */
  .cursor.N .arrow {
    left:50%; bottom:4px; transform:translateX(-50%);
    border-width: 0 6px 8px 6px; /* top right bottom left */
    border-color: transparent transparent var(--accent) transparent; /* ▲ up */
  }
  .cursor.S .arrow {
    left:50%; top:4px; transform:translateX(-50%);
    border-width: 8px 6px 0 6px;
    border-color: var(--accent) transparent transparent transparent; /* ▼ down */
  }
  .cursor.E .arrow {
    left:4px; top:50%; transform:translateY(-50%);
    border-width:6px 0 6px 8px;
    border-color: transparent transparent transparent var(--accent); /* ◄ left */
  }
  .cursor.W .arrow {
    right:4px; top:50%; transform:translateY(-50%);
    border-width:6px 8px 6px 0;
    border-color: transparent var(--accent) transparent transparent; /* ► right */
  }

  /* Cleared mark */
  .cleared::after {
    content:"/"; position:absolute; left:50%; top:50%; transform:translate(-50%,-50%) rotate(-12deg);
    font-weight:900; font-size:24px; color:#16a34a; text-shadow:0 0 6px rgba(22,163,74,.4);
  }

  /* Popup */
  .modal { position:fixed; inset:0; display:flex; align-items:flex-start; justify-content:center; background:rgba(0,0,0,.5); z-index:3; padding:20px 12px; }
  .card  { margin-top:12px; width:min(900px,96vw); background:var(--panel); color:var(--text);
           border:1px solid var(--border); border-radius:14px; padding:16px; }
  .row   { display:flex; gap:8px; align-items:center; flex-wrap:wrap; }
  .kbd   { font-family:ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
           background:color-mix(in oklab, var(--panel) 70%, var(--bg));
           padding:2px 6px; border-radius:6px; border:1px solid var(--border); }
  ul { margin:8px 0 0 18px; text-align:left; }
  .muted-block { color:var(--muted); margin-top:8px; }
  table { width:100%; border-collapse:collapse; margin-top:6px; }
  th, td { padding:6px 4px; border-bottom:1px solid var(--border); text-align:left; font-size:14px; }
</style>

<div class="stage">
  <div class="topbar">
    <div class="title">Task 2: Dot Test</div>
    <div class="row">
      <div class="muted">Time left: <b>{mmss(timeLeft)}</b></div>
      {#if showHelp}
        <button class="btn" type="button" on:click={backToHome}>Back</button>
        <button class="btn primary" type="button" on:click={startSession}>Start</button>
      {:else}
        <button class="btn" type="button" on:click={endSession}>End</button>
      {/if}
    </div>
  </div>

  <!-- Board (sized by JS to always fit) -->
  <div class="board-wrap" bind:this={wrapEl}>
    <div class="board" bind:this={boardEl} aria-label="Dot grid board">
      {#each board as row, r}
        {#each row as cell, c}
          <div class="cell {cell.cleared ? 'cleared' : ''}">
            <div class="inner">
              {#each cell.pts as p}
                <div class="dot" style="left:{p.x*100}%; top:{p.y*100}%"></div>
              {/each}
            </div>
            {#if r === curR && c === curC}
              <div class="cursor {face}">
                <div class="arrow"></div>
              </div>
            {/if}
          </div>
        {/each}
      {/each}
    </div>
  </div>

  {#if showHelp}
    <div class="modal" role="dialog" aria-modal="true">
      <div class="card">
        <div class="row" style="justify-content:space-between;">
          <h2 style="margin:0;">How it works</h2>
          <div class="row">
            <button class="btn" type="button" on:click={toggleTheme} title="Toggle light/dark">
              {theme === 'light' ? 'Dark Mode' : 'Light Mode'}
            </button>
            <button class="btn" type="button" on:click={backToHome}>Back</button>
          </div>
        </div>

        <!-- Options -->
        <div class="row" style="margin:8px 0;">
          <label for="tc">Number of <b>4-dot</b> boxes per board:</label>
          <select id="tc" bind:value={targetCount}>
            <option value={1}>1 (default)</option>
            <option value={2}>2</option>
            <option value={3}>3</option>
          </select>
        </div>

        <!-- Guidelines (left-aligned) -->
        <ul>
          <li>The board is a <b>5 × 10</b> grid. Dots appear scattered only inside the inner <b>60%</b> of each cell (no overlaps); the outer edge is blank.</li>
          <li>Exactly <b>{targetCount}</b> cell(s) contain <b>4 dots</b>. Find and mark them.</li>
          <li><b>Right hand:</b> <span class="kbd">I</span>=forward (up), <span class="kbd">M</span>=back (down), <span class="kbd">J</span>=turn left, <span class="kbd">L</span>=turn right.</li>
          <li><b>Left hand:</b> <span class="kbd">W</span>=mark the cell (adds “/”).</li>
          <li>Spawn: bottom-center, facing up. Left/Right change direction; Up/Down move.</li>
          <li>Max session: <b>3 minutes</b>. Clearing all 4-dot cells loads a new board instantly.</li>
        </ul>

        <div class="row" style="margin-top:12px;">
          <button class="btn primary" type="button" on:click={startSession}>Start</button>
          <button class="btn" type="button" on:click={clearLog}>Clear Records</button>
        </div>

        <!-- Summary (shown after End) -->
        {#if showSummary}
          <div class="muted-block">
            <h3 style="margin:12px 0 6px 0;">Session Summary</h3>
            <div>Boards cleared: <b>{boardsCleared}</b></div>
            <div>Targets per board: <b>{targetCount}</b> (total targets cleared: <b>{boardsCleared * targetCount}</b>)</div>
            <div>Duration: <b>{mmss(SESSION_MS - timeLeft)}</b></div>
          </div>
        {/if}

        <!-- Previous records (Top-5) -->
        <div class="muted-block">
          <h3 style="margin:12px 0 6px 0;">Previous Records (Top-5)</h3>
          {#if top5.length === 0}
            <div>No records yet.</div>
          {:else}
            <table>
              <thead><tr><th>When</th><th>Boards</th><th>Targets/Board</th><th>Duration</th></tr></thead>
              <tbody>
                {#each top5 as r}
                  <tr>
                    <td>{new Date(r.when).toLocaleString()}</td>
                    <td>{r.boards}</td>
                    <td>{r.targetsPerBoard}</td>
                    <td>{mmss(r.durationMs)}</td>
                  </tr>
                {/each}
              </tbody>
            </table>
          {/if}
        </div>
      </div>
    </div>
  {/if}
</div>
