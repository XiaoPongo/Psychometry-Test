<script lang="ts">
  import { onMount, onDestroy, createEventDispatcher } from 'svelte';
  const dispatch = createEventDispatcher();

  // ------- Config -------
  const SESSION_MS = 180_000;                    // 3 minutes
  const DISPLAY_TIMES = [600, 700, 800, 900];    // randomized per trial
  const GAP_MS = 400;

  const THEME_KEY = 'upmrc_theme';
  const LOG_KEY   = 'pattern_top5';

  // ------- State -------
  let showHelp = true;
  let showSummary = false;
  let timeLeft = SESSION_MS;
  let sessionStart = 0;
  let timerTick: number | null = null;

  let trials = 0;
  let correct = 0;

  // trial rendering
  type Family = 'stripes'|'triangles'|'hearts'|'squares'|'dots';
  type Contrast = 'high'|'low';
  let family: Family = 'stripes';
  let contrast: Contrast = 'high';
  let cards: { svg: string }[] = [];
  let oddIndex = 0;
  let windowOpen = false;
  let running = false;

  // pending timeouts for a trial (so End can cancel them)
  let tOpen: number | null = null;
  let tClose: number | null = null;
  let tGap: number | null = null;

  // theme
  let theme: 'light'|'dark' = 'light';
  function applyTheme(next:'light'|'dark'){ theme = next; document.documentElement.setAttribute('data-theme', next); localStorage.setItem(THEME_KEY, next); }
  function toggleTheme(){ applyTheme(theme==='light'?'dark':'light'); }

  // records
  type Rec = { when:number; durationMs:number; trials:number; correct:number; accuracy:number };
  function loadTop5(): Rec[] { try{const raw=localStorage.getItem(LOG_KEY);const arr=raw?JSON.parse(raw):[];return Array.isArray(arr)?arr:[]}catch{ return []; } }
  function persistTop5(r:Rec){ const arr = loadTop5(); arr.unshift(r); while(arr.length>5) arr.pop(); localStorage.setItem(LOG_KEY, JSON.stringify(arr)); }
  let top5: Rec[] = loadTop5();
  function clearLog(){ if (!confirm('Clear Pattern Test records?')) return; localStorage.removeItem(LOG_KEY); top5=[]; }

  // ------- Session -------
  function startSession(){
    showHelp=false; showSummary=false;
    trials=0; correct=0; timeLeft=SESSION_MS;
    sessionStart = Date.now(); running = true;
    nextTrial();
    if (timerTick) clearInterval(timerTick);
    timerTick = window.setInterval(()=> {
      if (!running) return;
      timeLeft = Math.max(0, timeLeft - 1000);
      if (timeLeft<=0) endSession();
    }, 1000) as unknown as number;
  }

  function endSession(){
    running = false;
    windowOpen = false;
    // clear any pending timeouts
    if (tOpen){ clearTimeout(tOpen); tOpen=null; }
    if (tClose){ clearTimeout(tClose); tClose=null; }
    if (tGap){ clearTimeout(tGap); tGap=null; }
    if (timerTick){ clearInterval(timerTick); timerTick=null; }

    const acc = trials ? Math.round(1000*(100*correct/trials))/10 : 0;
    const rec: Rec = { when: sessionStart||Date.now(), durationMs: Date.now()-sessionStart, trials, correct, accuracy: acc };
    persistTop5(rec); top5 = loadTop5();

    showHelp=true; showSummary=true;
  }

  // ------- Trial generation -------
  function nextTrial(){
    if (!running) return;

    windowOpen = false;

    family   = pick(['stripes','triangles','hearts','squares','dots']);
    contrast = pick(['high','low']);

    oddIndex = Math.floor(Math.random()*5);
    // build 5 svgs; ensure any <pattern> IDs are unique per card
    cards = Array.from({length:5}, (_,i)=>({ svg: svgFor(family, i===oddIndex, contrast, `card${Date.now()}_${i}`) }));

    const displayMs = pick(DISPLAY_TIMES);

    // Open window immediately (rendered already)
    tOpen = window.setTimeout(() => {
      if (!running) return;
      windowOpen = true;
      // Close after randomized duration
      tClose = window.setTimeout(() => {
        if (!running) return;
        if (windowOpen){
          windowOpen = false;
          trials++;
          // Missed; schedule next
          tGap = window.setTimeout(() => { if (running && timeLeft>0) nextTrial(); }, GAP_MS);
        }
      }, displayMs) as unknown as number;
    }, 0) as unknown as number;
  }

  // ------- Colors (no DOM reads before mount) -------
  function fgColor(c:Contrast){ return c==='high' ? (theme==='light'?'#111111':'#e8eaed') : (theme==='light'?'rgba(0,0,0,.45)':'rgba(255,255,255,.45)'); }
  function bgColor(){ return theme==='light' ? '#ffffff' : '#15161a'; }
  function strokeColor(c:Contrast){ return c==='high' ? (theme==='light'?'#111111':'#e8eaed') : (theme==='light'?'rgba(0,0,0,.45)':'rgba(255,255,255,.45)'); }

  // ------- SVG makers with *unique IDs* -------
  function svgFor(f:Family, odd:boolean, c:Contrast, uid:string): string {
    const fg = fgColor(c);
    const bg = bgColor();
    const st = strokeColor(c);

    if (f==='stripes'){
      // four share angleA/spacingA; odd uses angleB/spacingB
      const angle = odd ? 65 : 20;
      const step  = odd ? 12 : 16;
      return `
        <svg viewBox="0 0 120 120" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
          <defs>
            <pattern id="p_${uid}" patternUnits="userSpaceOnUse" width="${step}" height="${step}"
                     patternTransform="rotate(${angle})">
              <rect width="${step}" height="${step}" fill="${bg}"/>
              <rect width="${step/2}" height="${step}" fill="${fg}"/>
            </pattern>
          </defs>
          <rect x="10" y="10" width="100" height="100" rx="12" fill="url(#p_${uid})"/>
        </svg>`;
    }

    if (f==='triangles'){
      // four up, odd down (and slightly different scale)
      const up = !odd;
      const s = odd ? 0.9 : 1.05;
      const pointsUp   = '60,20 100,100 20,100';
      const pointsDown = '20,20 100,20 60,100';
      const pts = up ? pointsUp : pointsDown;
      return `
        <svg viewBox="0 0 120 120" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
          <rect x="8" y="8" width="104" height="104" rx="12" fill="${bg}"/>
          <g transform="translate(60,60) scale(${s}) translate(-60,-60)">
            <polygon points="${pts}" fill="${fg}"/>
            <polygon points="${translate(pts, -26,18)}" fill="${fg}"/>
            <polygon points="${translate(pts,  26,18)}" fill="${fg}"/>
          </g>
        </svg>`;
    }

    if (f==='hearts'){
      const flip = odd;
      const path = 'M60 95 C 20 70, 20 40, 40 30 C 55 25, 60 40, 60 40 C 60 40, 65 25, 80 30 C 100 40, 100 70, 60 95 Z';
      const tf = flip ? 'translate(120,120) rotate(180)' : '';
      return `
        <svg viewBox="0 0 120 120" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
          <rect x="8" y="8" width="104" height="104" rx="12" fill="${bg}"/>
          <g transform="${tf}">
            <path d="${path}" fill="${fg}"/>
          </g>
        </svg>`;
    }

    if (f==='squares'){
      // four rotated 15°, odd rotated -15°
      const rot = odd ? -15 : 15;
      return `
        <svg viewBox="0 0 120 120" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
          <rect x="8" y="8" width="104" height="104" rx="12" fill="${bg}"/>
          <g transform="translate(60,60) rotate(${rot}) translate(-60,-60)">
            <rect x="28" y="28" width="64" height="64" fill="none" stroke="${st}" stroke-width="10"/>
          </g>
        </svg>`;
    }

    // dots: four have dense grid, odd has sparse (or vice versa)
    const dense = !odd;
    const step = dense ? 16 : 24;
    return `
      <svg viewBox="0 0 120 120" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
        <defs>
          <pattern id="dp_${uid}" patternUnits="userSpaceOnUse" width="${step}" height="${step}">
            <rect width="${step}" height="${step}" fill="${bg}"/>
            <circle cx="${step/2}" cy="${step/2}" r="${dense?3:2}" fill="${fg}"/>
          </pattern>
        </defs>
        <rect x="10" y="10" width="100" height="100" rx="12" fill="url(#dp_${uid})"/>
      </svg>`;
  }

  function translate(points:string, dx:number, dy:number){
    return points.split(' ').map(p=>{
      const [x,y]=p.split(',').map(Number);
      return `${x+dx},${y+dy}`;
    }).join(' ');
  }
  function pick<T>(arr:T[]):T { return arr[Math.floor(Math.random()*arr.length)]; }

  // ------- Input (1–5) -------
  function onKey(e:KeyboardEvent){
    if (!running || !windowOpen) return;
    if (!/^[1-5]$/.test(e.key)) return;
    windowOpen = false;
    trials++;
    if ((+e.key - 1) === oddIndex) correct++;
    if (running && timeLeft>0){
      if (tGap) clearTimeout(tGap);
      tGap = window.setTimeout(() => nextTrial(), GAP_MS) as unknown as number;
    }
  }

  // ------- lifecycle -------
  onMount(()=>{
    window.addEventListener('keydown', onKey);
    const saved = (localStorage.getItem(THEME_KEY) as 'light'|'dark'|null) ?? 'light';
    applyTheme(saved);
  });
  onDestroy(()=>{
    window.removeEventListener('keydown', onKey);
    if (timerTick) clearInterval(timerTick);
    if (tOpen) clearTimeout(tOpen);
    if (tClose) clearTimeout(tClose);
    if (tGap) clearTimeout(tGap);
  });

  function goBack(){ endSession(); dispatch('back'); }
  function mmss(ms:number){ const s=Math.ceil(ms/1000), m=Math.floor(s/60), sec=s%60; return `${m}:${sec.toString().padStart(2,'0')}`; }
  function fmtPct(n:number){ return `${(Math.round(n*10)/10).toFixed(1)}%`; }
</script>

<style>
  :root { --bg:#0b0b0e; --panel:#15161a; --text:#e8eaed; --muted:#8a8f98; --border:#2a2d34; }
  :root[data-theme='light'] { --bg:#f7f8fb; --panel:#ffffff; --text:#0f172a; --muted:#5b6270; --border:#e5e7eb; }

  .stage { position:fixed; inset:5vh 5vw; display:flex; flex-direction:column; gap:10px; }
  .topbar { display:flex; justify-content:space-between; align-items:center; gap:10px;
            background:var(--panel); border:1px solid var(--border); border-radius:12px; padding:10px 14px; }
  .muted{ color:var(--muted); }
  .btn{ background:var(--panel); color:var(--text); border:1px solid var(--border); padding:8px 12px; border-radius:10px; cursor:pointer;}
  .btn.primary{ background:#1d4ed8; color:#fff; border:none; }

  .board { flex:1; display:grid; place-items:center; }
  .row5 { display:grid; grid-template-columns: repeat(5, minmax(0, 1fr)); gap:18px; width:min(1100px, 100%); }
  .cell { background:var(--panel); border:1px solid var(--border); border-radius:14px; display:grid; place-items:center; aspect-ratio:1/1; padding:8px; position:relative; }
  .num { position:absolute; top:8px; left:10px; font-size:12px; color:var(--muted); }

  /* Popup */
  .modal { position:fixed; inset:0; display:flex; align-items:flex-start; justify-content:center; background:rgba(0,0,0,.5); z-index:3; padding:20px 12px; }
  .card  { margin-top:12px; width:min(900px,96vw); background:var(--panel); color:var(--text); border:1px solid var(--border); border-radius:14px; padding:16px; }
  .row   { display:flex; gap:8px; align-items:center; flex-wrap:wrap; }
  .kbd { font-family: ui-monospace, Menlo, Consolas, monospace; background:color-mix(in oklab, var(--panel) 70%, var(--bg)); padding:2px 6px; border-radius:6px; border:1px solid var(--border); }
  ul { margin:8px 0 0 18px; text-align:left; }
  table { width:100%; border-collapse:collapse; margin-top:8px;}
  th,td { border-bottom:1px solid var(--border); text-align:left; padding:6px 4px; font-size:14px;}
</style>

<div class="stage">
  <div class="topbar">
    <div><strong>Task 4: Odd-One-Out</strong></div>
    <div class="row">
      <div class="muted">Time left: <b>{mmss(timeLeft)}</b></div>
      {#if showHelp}
        <button class="btn" on:click={goBack}>Back</button>
        <button class="btn primary" on:click={startSession}>Start</button>
      {:else}
        <button class="btn" on:click={endSession}>End</button>
      {/if}
    </div>
  </div>

  <div class="board" aria-live="polite">
    <div class="row5">
      {#each cards as c, i}
        <div class="cell">
          {@html c.svg}
          <div class="num">{i+1}</div>
        </div>
      {/each}
    </div>
  </div>

  {#if showHelp}
    <div class="modal" role="dialog" aria-modal="true">
      <div class="card">
        <div class="row" style="justify-content:space-between;">
          <h2 style="margin:0;">How this test works</h2>
          <div class="row">
            <button class="btn" on:click={toggleTheme}>{theme==='light'?'Dark Mode':'Light Mode'}</button>
            <button class="btn" on:click={goBack}>Back</button>
          </div>
        </div>

        <ul>
          <li>Five images appear at once for <b>600–900 ms</b> (random each trial). One image differs from the other four.</li>
          <li>Press <span class="kbd">1</span>–<span class="kbd">5</span> to select the odd image before time runs out.</li>
          <li>Pattern families rotate: stripes, triangles, hearts, rotated squares, and dots grid. Each may be high-contrast or low-visibility.</li>
          <li>Session length: <b>3 minutes</b>. Summary and top-5 records are saved locally.</li>
        </ul>

        <div class="row" style="margin-top:12px;">
          <button class="btn primary" on:click={startSession}>Start</button>
          <button class="btn" on:click={clearLog}>Clear Records</button>
        </div>

        {#if showSummary}
          <div style="margin-top:10px;">
            <h3 style="margin:0 0 6px 0;">Session Summary</h3>
            <div>Trials: <b>{trials}</b> &nbsp; Correct: <b>{correct}</b></div>
            <div>Duration: <b>{mmss(Date.now()-sessionStart)}</b></div>
          </div>
        {/if}

        <div style="margin-top:10px;">
          <h3 style="margin:0 0 6px 0;">Previous Records (Top-5)</h3>
          {#if top5.length===0}
            <div class="muted">No records yet.</div>
          {:else}
            <table>
              <thead><tr><th>When</th><th>Trials</th><th>Correct</th><th>Duration</th></tr></thead>
              <tbody>
                {#each top5 as r}
                <tr>
  <td>{new Date(r.when).toLocaleString()}</td>
  <td>{r.trials}</td>
  <td>{r.correct}</td>
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
