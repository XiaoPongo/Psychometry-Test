<script lang="ts">
  import { onMount } from 'svelte';
  import RYGTest from './RYGTest.svelte';
  import DotTest from './DotTest.svelte';
  import PatternOddOne from './PatternOddOne.svelte';

  type Task = 'home' | 'ryg' | 'dot' | 'brake' | 'pattern';
  let task: Task = 'home';

  let theme: 'light' | 'dark' = 'light';
  const THEME_KEY = 'upmrc_theme';

  function applyTheme(next: 'light' | 'dark') {
    theme = next;
    document.documentElement.setAttribute('data-theme', next);
    localStorage.setItem(THEME_KEY, next);
  }
  function toggleTheme() { applyTheme(theme==='light'?'dark':'light'); }

  function open(t: Task) { task = t; }
  function backHome() { task = 'home'; }

  onMount(() => {
    const saved = (localStorage.getItem(THEME_KEY) as 'light'|'dark'|null) ?? 'light';
    applyTheme(saved);
  });
</script>

<style>
  :root { --bg:#0b0b0e; --panel:#15161a; --text:#e8eaed; --muted:#8a8f98; --border:#2a2d34; }
  :root[data-theme='light'] { --bg:#f7f8fb; --panel:#ffffff; --text:#0f172a; --muted:#5b6270; --border:#e5e7eb; }

  :global(html), :global(body) { margin:0; background:var(--bg); color:var(--text); font-family: Inter, system-ui, -apple-system, Segoe UI, Roboto, sans-serif; }

  .wrap { max-width: 980px; margin: 32px auto; padding: 16px; }
  header { display:flex; align-items:center; justify-content:space-between; gap:12px; margin-bottom:18px; }
  h1 { margin:0; font-size:28px; }
  .muted { color:var(--muted); }

  .toolbar { display:flex; gap:8px; align-items:center; }
  .btn { background:#1d4ed8; color:white; border:none; padding:10px 14px; border-radius:10px; cursor:pointer; }
  .back { background:#2a2d34; color:var(--text); border:1px solid var(--border); }
  :root[data-theme='light'] .back { background:#f3f4f6; }
  .toggle { background: var(--panel); color: var(--text); border:1px solid var(--border); padding:8px 12px; border-radius:10px; cursor:pointer; }

  .grid { display:grid; grid-template-columns: repeat(2, minmax(0,1fr)); gap:16px; }
  .card { background:#0d0f14; border:1px solid var(--border); border-radius:14px; padding:18px; display:flex; flex-direction:column; gap:10px; }
  :root[data-theme='light'] .card { background:#ffffff; }
  .card h3 { margin:0; }
  .sep { height:1px; background:var(--border); margin:16px 0; }
</style>

{#if task === 'home'}
  <div class="wrap">
    <header>
      <div>
        <h1>Railway Psychometry Tools</h1>
        <div class="muted">Choose a test to begin</div>
      </div>
      <div class="toolbar">
        <button class="toggle" on:click={toggleTheme}>
          {theme === 'light' ? 'Switch to Dark' : 'Switch to Light'}
        </button>
      </div>
    </header>

    <div class="grid">
      <div class="card">
        <h3>Task 1: RYG Quick Light Test</h3>
        <p class="muted">Reaction / gate-based color test (W-hold + R/Y/G).</p>
        <button class="btn" on:click={() => open('ryg')}>Open</button>
      </div>

      <div class="card">
        <h3>Task 2: Dot Test</h3>
        <p class="muted">Find & cancel the 4-dot cells (I/J/L/M + W).</p>
        <button class="btn" on:click={() => open('dot')}>Open</button>
      </div>

      <div class="card">
        <h3>Task 3: Brake Test</h3>
        <p class="muted">Stimulus-to-brake reaction timing simulation.</p>
        <button class="btn" on:click={() => open('brake')}>Open</button>
      </div>

      <div class="card">
        <h3>Task 4: Odd-One-Out</h3>
        <p class="muted">Press 1–5 for the odd image within 600–900 ms.</p>
        <button class="btn" on:click={() => open('pattern')}>Open</button>
      </div>
    </div>

    <div class="sep"></div>
    <p class="muted">Tip: Use the Back button inside any test to return here.</p>
  </div>
{:else if task === 'ryg'}
  <div class="wrap">
    <div class="toolbar" style="margin-bottom:12px;">
      <button class="back" on:click={backHome}>← Back</button>
      <button class="toggle" on:click={toggleTheme}>{theme==='light'?'Switch to Dark':'Switch to Light'}</button>
    </div>
    <RYGTest />
  </div>
{:else if task === 'dot'}
  <div class="wrap" style="max-width:none; padding:0;">
    <div class="toolbar" style="padding:12px 16px;">
      <button class="back" on:click={backHome}>← Back</button>
      <button class="toggle" on:click={toggleTheme} style="margin-left:auto;">
        {theme==='light'?'Switch to Dark':'Switch to Light'}
      </button>
    </div>
    <DotTest on:back={backHome} />
  </div>
{:else if task === 'pattern'}
  <div class="wrap" style="max-width:none; padding:0;">
    <div class="toolbar" style="padding:12px 16px;">
      <button class="back" on:click={backHome}>← Back</button>
      <button class="toggle" on:click={toggleTheme} style="margin-left:auto;">
        {theme==='light'?'Switch to Dark':'Switch to Light'}
      </button>
    </div>
    <PatternOddOne on:back={backHome} />
  </div>
{:else if task === 'brake'}
  <div class="wrap">
    <div class="toolbar" style="margin-bottom:12px;">
      <button class="back" on:click={backHome}>← Back</button>
      <button class="toggle" on:click={toggleTheme}>{theme==='light'?'Switch to Dark':'Switch to Light'}</button>
    </div>
    <h2>Brake Test</h2>
    <p class="muted">Coming soon. (We’ll add this next.)</p>
  </div>
{/if}
