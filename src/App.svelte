<script>
  let audioCtx = null;
  let oscillator = null;
  let gainNode = null;
  let isPlaying = false;

  let frequency = 440;
  let volume = 50;
  let waveform = 'sine';
  let pulseMode = false;
  let pulseRate = 5;
  let dutyCycle = 50;
  let freqInput = '440';

  let theme = 'dark';

  import { onMount, onDestroy } from 'svelte';

  onMount(() => {
    const stored = localStorage.getItem('theme');
    const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
    theme = stored || (prefersDark ? 'dark' : 'light');
    document.documentElement.setAttribute('data-theme', theme);
  });

  function toggleTheme() {
    theme = theme === 'dark' ? 'light' : 'dark';
    document.documentElement.setAttribute('data-theme', theme);
    localStorage.setItem('theme', theme);
  }

  function freqToLog(val) {
    // Map 0-1 to 20-20000 Hz log scale
    return Math.round(20 * Math.pow(1000, val));
  }

  function logToFreq(val) {
    // Map 20-20000 Hz to 0-1 log scale
    return Math.log(val / 20) / Math.log(1000);
  }

  let sliderPos = logToFreq(440);

  function onSliderInput() {
    frequency = freqToLog(sliderPos);
    freqInput = String(frequency);
    if (isPlaying) updateOscillator();
  }

  function onFreqInput() {
    let val = parseInt(freqInput);
    if (isNaN(val)) val = 440;
    val = Math.max(20, Math.min(20000, val));
    frequency = val;
    sliderPos = logToFreq(val);
    freqInput = String(val);
    if (isPlaying) updateOscillator();
  }

  function startTone() {
    if (audioCtx) audioCtx.close();
    audioCtx = new (window.AudioContext || window.webkitAudioContext)();

    gainNode = audioCtx.createGain();
    gainNode.gain.value = volume / 100;
    gainNode.connect(audioCtx.destination);

    oscillator = audioCtx.createOscillator();
    oscillator.type = waveform;
    oscillator.frequency.value = frequency;
    oscillator.connect(gainNode);
    oscillator.start();
    isPlaying = true;

    if (pulseMode) startPulse();
  }

  let pulseInterval = null;

  function startPulse() {
    if (pulseInterval) clearInterval(pulseInterval);
    const period = 1000 / pulseRate;
    const onTime = period * (dutyCycle / 100);
    const offTime = period - onTime;

    function pulse() {
      if (!isPlaying) return;
      gainNode.gain.value = volume / 100;
      setTimeout(() => {
        if (isPlaying) gainNode.gain.value = 0;
      }, onTime);
    }
    pulse();
    pulseInterval = setInterval(pulse, period);
  }

  function stopTone() {
    isPlaying = false;
    if (pulseInterval) {
      clearInterval(pulseInterval);
      pulseInterval = null;
    }
    if (oscillator) {
      try { oscillator.stop(); } catch(e) {}
      oscillator = null;
    }
    if (gainNode) gainNode.disconnect();
    gainNode = null;
    if (audioCtx) {
      audioCtx.close();
      audioCtx = null;
    }
  }

  function updateOscillator() {
    if (oscillator && isPlaying) {
      oscillator.frequency.value = frequency;
      oscillator.type = waveform;
    }
  }

  function updateVolume() {
    if (gainNode && isPlaying) {
      gainNode.gain.value = pulseMode ? volume / 100 : volume / 100;
    }
  }

  function togglePlay() {
    if (isPlaying) stopTone();
    else startTone();
  }

  function onPulseToggle() {
    if (isPlaying) {
      stopTone();
      startTone();
    }
  }

  function onWaveformChange() {
    if (isPlaying) updateOscillator();
  }

  $: if (gainNode && isPlaying && !pulseMode) {
    gainNode.gain.value = volume / 100;
  }

  onDestroy(() => {
    stopTone();
  });
</script>

<div class="app-container">
  <!-- Header -->
  <div class="header">
    <h1>TONE GENERATOR</h1>
    <button class="theme-btn" on:click={toggleTheme} aria-label="Toggle theme">
      {#if theme === 'dark'}
        <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2">
          <circle cx="12" cy="12" r="4"></circle>
          <path d="M12 2v2M12 20v2M4.93 4.93l1.41 1.41M17.66 17.66l1.41 1.41M2 12h2M20 12h2M4.93 19.07l1.41-1.41M17.66 6.34l1.41-1.41"></path>
        </svg>
      {:else}
        <svg viewBox="0 0 24 24" width="16" height="16" fill="currentColor">
          <path d="M20.354 15.354A9 9 0 018.646 3.646 9 9 0 1012 21a8.96 8.96 0 008.354-5.646z"></path>
        </svg>
      {/if}
    </button>
  </div>

  <!-- Main Controls -->
  <div class="controls-panel">
    <!-- Frequency -->
    <div class="control-group">
      <label class="control-label">Frequency</label>
      <div class="freq-row">
        <input type="range" min="0" max="1" step="0.001" bind:value={sliderPos} on:input={onSliderInput} class="freq-slider" />
        <input type="number" bind:value={freqInput} on:change={onFreqInput} class="freq-input" min="20" max="20000" />
        <span class="unit">Hz</span>
      </div>
      <div class="freq-markers">
        <span>20</span><span>100</span><span>1k</span><span>10k</span><span>20k</span>
      </div>
    </div>

    <!-- Volume -->
    <div class="control-group">
      <label class="control-label">Volume</label>
      <div class="slider-row">
        <input type="range" min="0" max="100" bind:value={volume} on:input={updateVolume} />
        <span class="value-badge">{volume}%</span>
      </div>
    </div>

    <!-- Waveform -->
    <div class="control-group">
      <label class="control-label">Waveform</label>
      <div class="waveform-selector">
        {#each ['sine', 'square', 'sawtooth', 'triangle'] as wf}
          <button
            class="wave-btn"
            class:active={waveform === wf}
            on:click={() => { waveform = wf; onWaveformChange(); }}
          >
            {wf}
          </button>
        {/each}
      </div>
    </div>

    <!-- Pulse Mode Toggle -->
    <div class="control-group">
      <label class="control-label">
        <input type="checkbox" bind:checked={pulseMode} on:change={onPulseToggle} class="toggle-checkbox" />
        Pulse Mode
      </label>
      {#if pulseMode}
        <div class="pulse-controls">
          <div class="sub-control">
            <label>Pulse Rate</label>
            <div class="slider-row">
              <input type="range" min="0.5" max="50" step="0.5" bind:value={pulseRate} on:input={onPulseToggle} />
              <span class="value-badge">{pulseRate} Hz</span>
            </div>
          </div>
          <div class="sub-control">
            <label>Duty Cycle</label>
            <div class="slider-row">
              <input type="range" min="5" max="95" step="1" bind:value={dutyCycle} on:input={onPulseToggle} />
              <span class="value-badge">{dutyCycle}%</span>
            </div>
          </div>
        </div>
      {/if}
    </div>

    <!-- Play Button -->
    <button class="play-btn" class:playing={isPlaying} on:click={togglePlay}>
      {isPlaying ? '■ STOP' : '▶ PLAY'}
    </button>
  </div>
</div>

<style>
  .app-container {
    max-width: 640px;
    margin: 0 auto;
    padding: 0 20px 40px;
  }

  .header {
    display: flex;
    align-items: center;
    justify-content: center;
    position: relative;
    padding: 16px 0;
    border-bottom: 1px solid var(--border-color);
    margin-bottom: 32px;
  }

  h1 {
    font-size: 1.1rem;
    font-weight: 700;
    letter-spacing: 0.05em;
    color: var(--text-primary);
  }

  .theme-btn {
    background: none;
    border: 1px solid var(--border-color);
    border-radius: 999px;
    width: 32px;
    height: 32px;
    padding: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    color: var(--text-primary);
    cursor: pointer;
    margin-left: 0;
    position: absolute;
    right: 0;
    top: 50%;
    transform: translateY(-50%);
  }
  .theme-btn svg {
    width: 16px;
    height: 16px;
    flex-shrink: 0;
    display: block;
  }
  .theme-btn:hover { border-color: var(--accent); }

  .controls-panel {
    display: flex;
    flex-direction: column;
    gap: 28px;
  }

  .control-group {
    background: var(--surface-1);
    border: 1px solid var(--border-color);
    border-radius: 12px;
    padding: 20px;
  }

  .control-label {
    font-size: 0.8rem;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.08em;
    color: var(--text-secondary);
    margin-bottom: 12px;
    display: flex;
    align-items: center;
    gap: 8px;
  }

  .freq-row {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  .freq-slider {
    flex: 1;
  }

  .freq-input {
    width: 90px;
    background: var(--surface-2);
    color: var(--text-primary);
    border: 1px solid var(--border-color);
    border-radius: 8px;
    padding: 8px 10px;
    font-size: 1rem;
    text-align: center;
    outline: none;
  }
  .freq-input:focus { border-color: var(--accent); }

  .unit {
    font-size: 0.85rem;
    color: var(--text-secondary);
    font-weight: 500;
  }

  .freq-markers {
    display: flex;
    justify-content: space-between;
    margin-top: 6px;
    font-size: 0.65rem;
    color: var(--text-secondary);
    padding: 0 2px;
  }

  .slider-row {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  .slider-row input[type="range"] { flex: 1; }

  .value-badge {
    font-size: 0.85rem;
    font-weight: 600;
    color: var(--accent);
    min-width: 48px;
    text-align: right;
    font-variant-numeric: tabular-nums;
  }

  .waveform-selector {
    display: flex;
    gap: 6px;
    flex-wrap: wrap;
  }

  .wave-btn {
    flex: 1;
    min-width: 70px;
    padding: 10px 8px;
    background: var(--surface-2);
    color: var(--text-primary);
    border: 1px solid var(--border-color);
    border-radius: 8px;
    font-size: 0.8rem;
    font-weight: 500;
    text-transform: capitalize;
    transition: all 0.15s;
  }

  .wave-btn:hover {
    border-color: var(--accent);
  }

  .wave-btn.active {
    background: var(--accent);
    color: white;
    border-color: var(--accent);
  }

  .toggle-checkbox {
    width: 18px;
    height: 18px;
    cursor: pointer;
    accent-color: var(--accent);
  }

  .pulse-controls {
    display: flex;
    flex-direction: column;
    gap: 16px;
    margin-top: 12px;
    padding-top: 12px;
    border-top: 1px solid var(--border-color);
  }

  .sub-control label {
    font-size: 0.75rem;
    color: var(--text-secondary);
    display: block;
    margin-bottom: 6px;
  }

  .play-btn {
    width: 100%;
    padding: 16px;
    font-size: 1rem;
    font-weight: 700;
    letter-spacing: 0.05em;
    border-radius: 12px;
    background: var(--accent);
    color: white;
    border: none;
    transition: all 0.2s;
  }

  .play-btn:hover {
    background: var(--accent-hover);
  }

  .play-btn.playing {
    background: var(--danger);
  }

  .play-btn.playing:hover {
    background: #c82333;
  }

  @media (max-width: 480px) {
    .freq-input { width: 72px; }
    .wave-btn { min-width: 60px; font-size: 0.75rem; }
  }
</style>
