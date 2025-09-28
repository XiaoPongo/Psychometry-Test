<script lang="ts">
  import { onMount, onDestroy } from 'svelte';

  // --- Types ---
  type Color = 'R' | 'Y' | 'G';
  type Phase = 'idle' | 'armed' | 'awaiting_L' | 'stimulus_on' | 'finished';

  type Trial = {
    color: Color;
    onset: number;          // performance.now()
    respondedAt?: number;   // first keydown time
    rt?: number;            // respondedAt - onset (ms), only for first response
    correct: boolean;       // correct first response?
    premature: boolean;     // pressed R/G/Y while default before this stimulus
    wrongKey?: string;      // if wrong, which key
    missed?: boolean;       // no response within max RT window
  };

  type SessionSummary = {
    startedAt: number;
    durationMs: number;
    trials: Trial[];
    stats: OverallStats;
  };

  type OverallStats = {
    totalTrials: number;
    completedTrials: number; // stimuli actually shown
    correct: number;
    accuracy: number; // %
    prematureErrors: number;
    wrongKeyErrors: number;
    misses: number;
    overallAvgRT?: number; // ms, correct only
    overallMedianRT?: number;
    perColor: Record<Color, {
      attempts: number;
      correct: number;
      accuracy: number;
      avgRT?: number;
      medianRT?: number;
      minRT?: number;
      maxRT?: number;
    }>;
    rtSamples: number[]; // correct-only RTs for histogram (ms)
    rtSamplesByColor: Record<Color, number[]>; // correct-only
  };

  // --- Config ---
  const TEST_DURATION_MS = 60_000;
  const L_HOLD_MS = 200;
  const MAX_RT_MS = 5000; // 5 seconds per input
  const KEY_DEBOUNCE_MS = 100;
  const STORAGE_KEY = 'upmrc_psycho_top5';
  const THEME_KEY = 'upmrc_theme';

  // --- Reactive state ---
  let phase: Phase = 'idle';
  let timeLeftMs = TEST_DURATION_MS;
  let testEndsAt = 0;
  let sessionStartWall = 0; // Date.now() at start
  let timerId: number | null = null;

  let trials: Trial[] = [];
  let currentTrial: Trial | null = null;

  // Gating with L
  let lIsDown = false;
  let lDownAt = 0;
  let lGateTimer: number | null = null;
  let lastColor: Color | null = null;

  // Input bookkeeping
  let lastResponseAt = 0;
  let hasFocus = true;

  // Summary modal
  let showSummary = false;
  let summary: SessionSummary | null = null;

  // Top-5 log
  let top5: SessionSummary[] = loadTop5();

  // Premature flag for next trial
  let flagPrematureForNext = false;

  // Histogram cache (reactive)
  let hist: { edges: number[]; counts: number[]; max: number } | null = null;
  $: hist = summary ? histogram(summary.stats.rtSamples, 10) : null;

  // Theme (data-theme on <html>)
  let lightMode = false;
  function applyTheme() {
    const root = document.documentElement;
    root.setAttribute('data-theme', lightMode ? 'light' : 'dark');
    localStorage.setItem(THEME_KEY, lightMode ? 'light' : 'dark');
  }
  function toggleTheme() { lightMode = !lightMode; applyTheme(); }

  // --- Lifecycle ---
  onMount(() => {
    // Init theme
    try {
      const saved = localStorage.getItem(THEME_KEY);
      lightMode = saved === 'light';
      applyTheme();
    } catch {}

    const kd = (e: KeyboardEvent) => handleKeyDown(e);
    const ku = (e: KeyboardEvent) => handleKeyUp(e);
    const onF = () => hasFocus = true;
    const onB = () => hasFocus = false;
    window.addEventListener('keydown', kd);
    window.addEventListener('keyup', ku);
    window.addEventListener('focus', onF);
    window.addEventListener('blur', onB);
    return () => {
      window.removeEventListener('keydown', kd);
      window.removeEventListener('keyup', ku);
      window.removeEventListener('focus', onF);
      window.removeEventListener('blur', onB);
      clearTimer();
      if (lGateTimer) clearTimeout(lGateTimer);
    };
  });

  onDestroy(() => { /* cleanup handled in onMount return */ });

  // --- Helpers ---
  function pickNextColor(): Color {
    const colors: Color[] = ['R', 'Y', 'G'];
    const candidates = lastColor ? colors.filter(c => c !== lastColor) : colors;
    const i = Math.floor(Math.random() * candidates.length);
    return candidates[i];
  }

  function startTest() {
    if (phase !== 'idle') return;
    trials = [];
    currentTrial = null;
    showSummary = false;
    summary = null;
    sessionStartWall = Date.now();

    phase = 'armed';
    const now = performance.now();
    testEndsAt = now + TEST_DURATION_MS;
    timeLeftMs = TEST_DURATION_MS;

    // Start ticking
    timerId = window.setInterval(() => {
      const t = performance.now();
      timeLeftMs = Math.max(0, Math.round(testEndsAt - t));
      if (t >= testEndsAt) {
        clearTimer();
        if (phase === 'stimulus_on') {
          setTimeout(() => endTest(), MAX_RT_MS + 50);
        } else {
          endTest();
        }
      }
    }, 16) as unknown as number;

    phase = 'awaiting_L';
  }

  function clearTimer() {
    if (timerId) {
      clearInterval(timerId);
      timerId = null;
    }
  }

  function scheduleStimulusAfterLGate() {
    if (phase !== 'awaiting_L') return;
    if (lGateTimer) clearTimeout(lGateTimer);
    const now = performance.now();
    const alreadyHeld = now - lDownAt;
    const waitMs = Math.max(1, L_HOLD_MS - alreadyHeld);
    lGateTimer = window.setTimeout(() => {
      if (lIsDown && phase === 'awaiting_L' && performance.now() < testEndsAt) {
        showNextStimulus();
      }
    }, waitMs) as unknown as number;
  }

  function showNextStimulus() {
    if (performance.now() >= testEndsAt) return; // time’s up
    const color = pickNextColor();
    lastColor = color;
    currentTrial = {
      color,
      onset: performance.now(),
      correct: false,
      premature: flagPrematureForNext,
    };
    flagPrematureForNext = false;
    trials.push(currentTrial);
    phase = 'stimulus_on';

    const myTrial = currentTrial;
    window.setTimeout(() => {
      if (phase === 'stimulus_on' && currentTrial === myTrial && !myTrial.respondedAt) {
        myTrial.missed = true;
        phase = 'awaiting_L';
      }
    }, MAX_RT_MS + 1);
  }

  function endTest() {
    if (phase === 'finished') return;
    clearTimer();
    phase = 'finished';
    const stats = computeStats(trials);
    summary = {
      startedAt: sessionStartWall,
      durationMs: TEST_DURATION_MS,
      trials,
      stats
    };
    persistTop5(summary);
    top5 = loadTop5();
    showSummary = true;
  }

  // --- Stats & charts ---
  function computeStats(trs: Trial[]): OverallStats {
    const perColorInit = () => ({
      attempts: 0, correct: 0, accuracy: 0,
      avgRT: undefined as number | undefined,
      medianRT: undefined as number | undefined,
      minRT: undefined as number | undefined,
      maxRT: undefined as number | undefined
    });
    const perColor: Record<Color, any> = { R: perColorInit(), Y: perColorInit(), G: perColorInit() };

    const rtAllCorrect: number[] = [];
    const rtByColor: Record<Color, number[]> = { R: [], Y: [], G: [] };

    let prematureErrors = 0;
    let wrongKeyErrors = 0;
    let misses = 0;
    let completedTrials = 0;

    for (const t of trs) {
      completedTrials++;
      const p = perColor[t.color];
      p.attempts += 1;
      if (t.premature) prematureErrors += 1;
      if (t.missed) { misses += 1; continue; }

      if (t.respondedAt != null) {
        if (t.correct) {
          p.correct += 1;
          rtAllCorrect.push(t.rt!);
          rtByColor[t.color].push(t.rt!);
        } else {
          wrongKeyErrors += 1;
        }
      } else {
        misses += 1;
      }
    }

    const overallAvgRT = avg(rtAllCorrect);
    const overallMedianRT = median(rtAllCorrect);

    (['R','Y','G'] as Color[]).forEach(c => {
      const p = perColor[c];
      const arr = rtByColor[c];
      p.accuracy = p.attempts ? round1(100 * p.correct / p.attempts) : 0;
      if (arr.length) {
        p.avgRT = round1(avg(arr));
        p.medianRT = round1(median(arr));
        p.minRT = Math.min(...arr);
        p.maxRT = Math.max(...arr);
      }
    });

    const totalTrials = trs.length;
    const correct = rtAllCorrect.length;
    const accuracy = totalTrials ? round1(100 * correct / totalTrials) : 0;

    return {
      totalTrials,
      completedTrials,
      correct,
      accuracy,
      prematureErrors,
      wrongKeyErrors,
      misses,
      overallAvgRT: rtAllCorrect.length ? round1(overallAvgRT) : undefined,
      overallMedianRT: rtAllCorrect.length ? round1(overallMedianRT) : undefined,
      perColor,
      rtSamples: rtAllCorrect,
      rtSamplesByColor: rtByColor
    };
  }

  function avg(arr: number[]) { return arr.length ? arr.reduce((a,b)=>a+b,0)/arr.length : NaN; }
  function median(arr: number[]) {
    if (!arr.length) return NaN;
    const s = [...arr].sort((a,b)=>a-b);
    const m = Math.floor(s.length/2);
    return s.length%2? s[m] : (s[m-1]+s[m])/2;
  }
  function round1(x: number) { return Math.round(x*10)/10; }

  // Histogram bins 0..MAX_RT_MS
  function histogram(rt: number[], bins = 10) {
    const step = MAX_RT_MS / bins;
    const edges = Array.from({length: bins+1}, (_,i)=> i*step);
    const counts = Array.from({length: bins}, ()=>0);
    for (const v of rt) {
      let idx = Math.floor(v / step);
      if (idx < 0) idx = 0;
      if (idx >= bins) idx = bins-1;
      counts[idx]++;
    }
    return { edges, counts, max: Math.max(1, ...counts) };
  }

  // --- Input handling ---
  function handleKeyDown(e: KeyboardEvent) {
    if (!hasFocus) return;

    const key = e.key.toUpperCase();
    const now = performance.now();

    if (key === 'L') {
      if (!lIsDown) {
        lIsDown = true;
        lDownAt = now;
        if (phase === 'awaiting_L' && now < testEndsAt) {
          scheduleStimulusAfterLGate();
        }
      }
      return;
    }

    if (now - lastResponseAt < KEY_DEBOUNCE_MS) return;

    if (phase === 'awaiting_L') {
      if (key === 'R' || key === 'Y' || key === 'G') {
        flagPrematureForNext = true;
      }
      return;
    }

    if (phase === 'stimulus_on' && currentTrial) {
      const expected = currentTrial.color;
      const isLetter = /^[A-Z]$/.test(key);
      if (isLetter) {
        lastResponseAt = now;
        if (!currentTrial.respondedAt) {
          currentTrial.respondedAt = now;
          currentTrial.rt = now - currentTrial.onset;
          currentTrial.correct = (key === expected);
          if (!currentTrial.correct) {
            currentTrial.wrongKey = key; // logs non-R/Y/G or wrong R/Y/G
          }
          phase = 'awaiting_L';
        }
      }
    }
  }

  function handleKeyUp(e: KeyboardEvent) {
    const key = e.key.toUpperCase();
    const now = performance.now();
    if (key === 'L') {
      if (lGateTimer) {
        const held = now - lDownAt;
        if (held < L_HOLD_MS) {
          clearTimeout(lGateTimer);
          lGateTimer = null;
        }
      }
      lIsDown = false;
    }
  }

  // --- Persistence ---
  function loadTop5(): SessionSummary[] {
    try {
      const raw = localStorage.getItem(STORAGE_KEY);
      if (!raw) return [];
      const arr = JSON.parse(raw);
      return Array.isArray(arr) ? arr : [];
    } catch { return []; }
  }
  function persistTop5(s: SessionSummary) {
    const arr = loadTop5();
    arr.unshift(s);
    while (arr.length > 5) arr.pop();
    localStorage.setItem(STORAGE_KEY, JSON.stringify(arr));
  }
  function clearLog() {
    if (!confirm('Clear saved Top-5 attempts?')) return;
    localStorage.removeItem(STORAGE_KEY);
    top5 = [];
  }

  function refreshTest() {
    clearTimer();
    phase = 'idle';
    timeLeftMs = TEST_DURATION_MS;
    trials = [];
    currentTrial = null;
    lastColor = null;
    lIsDown = false;
    lDownAt = 0;
    if (lGateTimer) { clearTimeout(lGateTimer); lGateTimer = null; }
    showSummary = false;
    summary = null;
  }

  // --- Formatters ---
  function fmtMs(ms?: number) {
    if (ms == null || isNaN(ms)) return '—';
    return `${Math.round(ms)} ms`;
  }
  function fmtPct(p: number) {
    if (isNaN(p)) return '—';
    return `${(Math.round(p*10)/10).toFixed(1)}%`;
  }
  function mmss(ms: number) {
    const s = Math.ceil(ms/1000);
    const m = Math.floor(s/60);
    const sec = s % 60;
    return `${m}:${sec.toString().padStart(2,'0')}`;
  }

  // --- Display helpers ---
  function colorLabel(c?: Color) {
    if (!c) return '—';
    if (c === 'R') return 'RED';
    if (c === 'Y') return 'YELLOW';
    return 'GREEN';
  }
</script>

<style>
  /* THEMES */
  :root { --bg:#0b0b0e; --panel:#15161a; --muted:#8a8f98; --good:#2ecc71; --warn:#f1c40f; --bad:#e74c3c; --text:#e8eaed; --accent:#7aa2ff; --border:#2a2d34; --pad-off:#0a0b0f; }
  :root[data-theme='light'] { --bg:#f7f8fb; --panel:#ffffff; --muted:#5b6270; --good:#1f8f4a; --warn:#b8860b; --bad:#c0392b; --text:#0f172a; --accent:#1d4ed8; --border:#e5e7eb; --pad-off:#f3f4f6; }

  * { box-sizing: border-box; }
  body, :global(html), :global(body) { margin:0; background:var(--bg); color:var(--text); font-family: Inter, system-ui, -apple-system, Segoe UI, Roboto, sans-serif; }
  .wrap { max-width: 900px; margin: 24px auto; padding: 16px; }
  .topbar { display:flex; align-items:center; gap:12px; justify-content:space-between; }
  .buttons { display:flex; gap:8px; align-items:center; }
  button { background: var(--panel); color: var(--text); border: 1px solid var(--border); padding: 10px 14px; border-radius: 10px; cursor: pointer; }
  button.primary { background: linear-gradient(180deg,#2b6cf6,#1d4ed8); border:none; color:white; }
  :root[data-theme='light'] button.primary { background: linear-gradient(180deg,#60a5fa,#2563eb); }
  button:disabled { opacity: .6; cursor:not-allowed; }
  .status { display:flex; gap:16px; align-items:center; font-size:14px; color:var(--muted); }
  .stage { padding:2px 8px; border-radius: 8px; background:color-mix(in oklab, var(--panel) 80%, var(--bg)); }
  .timebig { font-weight:900; font-size:48px; letter-spacing:1px; color: var(--text); }
  .arena { margin-top: 18px; background: color-mix(in oklab, var(--panel) 90%, var(--bg)); border:1px solid var(--border); border-radius: 14px; padding: 24px; display:grid; gap:18px; }

  /* Single Light Pad */
  .lightpad { width:200px; height:200px; border-radius: 18px; border:1px solid var(--border); background: var(--pad-off); display:grid; place-items:center; margin: 0 auto; }
  .lightpad .label { font-weight:800; letter-spacing: .5px; text-transform: uppercase; }
  .lightpad.on.R { background:#b01515; box-shadow: 0 0 28px #f33; color:white; }
  .lightpad.on.Y { background:#caa314; box-shadow: 0 0 28px #ffdb4d; color:black; }
  .lightpad.on.G { background:#1f9a42; box-shadow: 0 0 28px #60ff8c; color:white; }

  .instructions { background: var(--panel); border:1px solid var(--border); border-radius: 12px; padding:14px; line-height:1.6; text-align:left; }
  .instructions h3 { margin: 6px 0 8px; }
  .instructions ul { margin: 6px 0 0 18px; list-style: disc; }
  .instructions li { margin: 6px 0; }
  .kbd { font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace; background:color-mix(in oklab, var(--panel) 70%, var(--bg)); padding:2px 6px; border-radius:6px; border:1px solid var(--border); }

  .overlay { position:fixed; inset:0; background:rgba(0,0,0,.7); display:flex; align-items:center; justify-content:center; z-index:2; color:white; font-size:18px; text-align:center; padding:24px; }
  .summary-modal { position:fixed; inset:0; display:flex; align-items:center; justify-content:center; background:rgba(0,0,0,.55); z-index:3; }
  .card { background:var(--panel); border:1px solid var(--border); border-radius: 14px; padding: 18px; width:min(900px,96vw); max-height: 90vh; overflow:auto; }
  .grid { display:grid; grid-template-columns: repeat(3,1fr); gap:12px; }
  .grid2 { display:grid; grid-template-columns: repeat(2,1fr); gap:12px; }
  .stat { background:color-mix(in oklab, var(--panel) 85%, var(--bg)); border:1px solid var(--border); border-radius:12px; padding:12px; }
  .muted { color:var(--muted); }
  .logs { margin-top: 16px; }
  table { width:100%; border-collapse: collapse; }
  th, td { border-bottom: 1px solid var(--border); padding:8px 6px; font-size: 14px; text-align:left; }
  .chart { width:100%; height:160px; }
  .chart rect { fill: var(--accent); }
  .tag { font-size:12px; color:#6b7280; }
</style>

<div class="wrap">
  <div class="topbar">
    <div class="status">
      <div class="stage">Phase: {phase}</div>
      <div class="timebig" aria-live="polite">{mmss(timeLeftMs)}</div>
    </div>
    <div class="buttons">
      <button class="primary" on:click={startTest} disabled={phase!=='idle'}>Start</button>
      <button on:click={refreshTest}>Refresh Test</button>
      <button on:click={clearLog} title="Clear saved Top-5">Clear Log</button>
      <button on:click={toggleTheme} aria-pressed={lightMode} title="Toggle light/dark">
        {lightMode ? 'Dark Mode' : 'Light Mode'}
      </button>
    </div>
  </div>

  <div class="arena">
    <!-- Single light -->
    <div class="lightpad {phase==='stimulus_on' ? 'on ' + (currentTrial?.color || '') : ''}" aria-live="polite" aria-label="Stimulus light">
      <div class="label">{phase==='stimulus_on' ? colorLabel(currentTrial?.color) : 'PRESS L KEY'}</div>
    </div>

    <!-- Clear, readable instructions -->
    <div class="instructions">
      <h3>How this test works</h3>
      <ul>
        <li><strong>Start</strong> the test, then hold <span class="kbd">L</span> for at least <strong>200 ms</strong> to reveal a color.</li>
        <li>Respond with the matching key: <span class="kbd">R</span> = RED, <span class="kbd">Y</span> = YELLOW, <span class="kbd">G</span> = GREEN.</li>
        <li>After every response, hold <span class="kbd">L</span> again (≥200 ms) to show the next color. Until then, the light shows <strong>PRESS L KEY</strong>.</li>
        <li>Pressing any other key while a color is shown is an <strong>error</strong>. Premature presses (R/Y/G while default) are also errors.</li>
        <li>Max reaction window is <strong>{MAX_RT_MS} ms</strong>. If you don’t respond in time, the light returns to default and it counts as a miss.</li>
        <li>The session lasts <strong>60 seconds</strong>. A summary appears when it ends.</li>
      </ul>
    </div>

    <div class="logs">
      <h3>Top-5 Attempts (local)</h3>
      {#if top5.length === 0}
        <div class="tag">No attempts saved yet.</div>
      {:else}
        <table>
          <thead>
            <tr>
              <th>#</th>
              <th>When</th>
              <th>Trials</th>
              <th>Accuracy</th>
              <th>Avg RT</th>
              <th>R / Y / G Avg RT</th>
            </tr>
          </thead>
          <tbody>
            {#each top5 as s, i}
              <tr>
                <td>{i+1}</td>
                <td>{new Date(s.startedAt).toLocaleString()}</td>
                <td>{s.stats.totalTrials}</td>
                <td>{fmtPct(s.stats.accuracy)}</td>
                <td>{fmtMs(s.stats.overallAvgRT)}</td>
                <td>
                  {fmtMs(s.stats.perColor.R.avgRT)} /
                  {fmtMs(s.stats.perColor.Y.avgRT)} /
                  {fmtMs(s.stats.perColor.G.avgRT)}
                </td>
              </tr>
            {/each}
          </tbody>
        </table>
      {/if}
    </div>
  </div>
</div>

{#if !hasFocus}
  <div class="overlay" on:click={() => window.focus()}>
    <div>
      <div style="font-size:22px; margin-bottom:10px;">Click to enable keyboard</div>
      This test uses keys: <span class="kbd">L</span> (gate), <span class="kbd">R</span>, <span class="kbd">Y</span>, <span class="kbd">G</span>.
    </div>
  </div>
{/if}

{#if showSummary && summary}
  <div class="summary-modal">
    <div class="card">
      <h2 style="margin-top:0;">Session Summary</h2>
      <div class="grid">
        <div class="stat">
          <div class="muted">Trials</div>
          <div style="font-size:22px; font-weight:700;">{summary.stats.totalTrials}</div>
        </div>
        <div class="stat">
          <div class="muted">Accuracy</div>
          <div style="font-size:22px; font-weight:700;">{fmtPct(summary.stats.accuracy)}</div>
        </div>
        <div class="stat">
          <div class="muted">Avg RT (correct)</div>
          <div style="font-size:22px; font-weight:700;">{fmtMs(summary.stats.overallAvgRT)}</div>
        </div>
      </div>

      <div class="grid2" style="margin-top:12px;">
        <div class="stat">
          <div class="muted">Errors</div>
          <div>Premature: <strong>{summary.stats.prematureErrors}</strong></div>
          <div>Wrong key: <strong>{summary.stats.wrongKeyErrors}</strong></div>
          <div>Misses: <strong>{summary.stats.misses}</strong></div>
        </div>
        <div class="stat">
          <div class="muted">Median RT (correct)</div>
          <div style="font-size:18px;">
            Overall: <strong>{fmtMs(summary.stats.overallMedianRT)}</strong><br>
            R / Y / G: <strong>{fmtMs(summary.stats.perColor.R.medianRT)}</strong> /
            <strong>{fmtMs(summary.stats.perColor.Y.medianRT)}</strong> /
            <strong>{fmtMs(summary.stats.perColor.G.medianRT)}</strong>
          </div>
        </div>
      </div>

      <!-- Histogram -->
      <div class="stat" style="margin-top:12px;">
        <div class="muted" style="margin-bottom:6px;">Reaction Time Histogram (correct only, 0–{MAX_RT_MS} ms)</div>
        {#if hist}
          <svg class="chart" viewBox="0 0 600 160" preserveAspectRatio="none" aria-label="RT histogram">
            {#each hist.counts as c, i}
              <rect x={(i * (600 / hist.counts.length)) + 2}
                    y={150 - ((hist.max ? (c / hist.max) * 140 : 0))}
                    width={(600 / hist.counts.length) - 4}
                    height={(hist.max ? (c / hist.max) * 140 : 0)}
                    rx="4" />
              <text x={(i * (600 / hist.counts.length)) + (600 / hist.counts.length) / 2}
      y="158" text-anchor="middle" font-size="10" fill="white">
  {Math.round(i * (MAX_RT_MS / hist.counts.length))}
</text>
            {/each}
          </svg>
        {:else}
          <div class="tag">No correct responses yet.</div>
        {/if}
      </div>

      <!-- Per-color stats -->
      <div class="grid" style="margin-top:12px;">
        {#each (['R','Y','G'] as Color[]) as c}
          <div class="stat">
            <div class="muted">Color {c}</div>
            <div>Attempts: <strong>{summary.stats.perColor[c].attempts}</strong></div>
            <div>Correct: <strong>{summary.stats.perColor[c].correct}</strong> ({fmtPct(summary.stats.perColor[c].accuracy)})</div>
            <div>Avg / Med RT: <strong>{fmtMs(summary.stats.perColor[c].avgRT)}</strong> / <strong>{fmtMs(summary.stats.perColor[c].medianRT)}</strong></div>
            <div>Min / Max RT: <strong>{fmtMs(summary.stats.perColor[c].minRT)}</strong> / <strong>{fmtMs(summary.stats.perColor[c].maxRT)}</strong></div>
          </div>
        {/each}
      </div>

      <div style="display:flex; gap:8px; justify-content:flex-end; margin-top:12px;">
        <button on:click={() => { showSummary = false; }}>Close</button>
        <button class="primary" on:click={() => { showSummary = false; refreshTest(); }}>New Test</button>
      </div>
    </div>
  </div>
{/if}
