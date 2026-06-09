<script>
  let audioCtx = null;
  let masterGain = null;

  // === Tone Generator State ===
  let isPlaying = false;
  let frequency = 440;
  let volume = 50;
  let waveform = 'sine';
  let pulseMode = false;
  let pulseRate = 5;
  let dutyCycle = 50;
  let freqInput = '440';
  let theme = 'dark';

  // === New: Sound Library / Presets ===
  let activeTab = 'tone';
  let customSamples = [];

  // === New: Effects ===
  let filterFreq = 20000;
  let filterRes = 0;
  let delayTime = 0;
  let delayFeedback = 0.3;
  let filterOn = false;
  let delayOn = false;

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
    return Math.round(20 * Math.pow(1000, val));
  }
  function logToFreq(val) {
    return Math.log(val / 20) / Math.log(1000);
  }
  let sliderPos = logToFreq(440);

  function onSliderInput() {
    frequency = freqToLog(sliderPos);
    freqInput = String(frequency);
    if (isPlaying) updateFrequency();
  }
  function onFreqInput() {
    let val = parseInt(freqInput);
    if (isNaN(val)) val = 440;
    val = Math.max(20, Math.min(20000, val));
    frequency = val;
    sliderPos = logToFreq(val);
    freqInput = String(val);
    if (isPlaying) updateFrequency();
  }

  // === Audio Graph Builder ===
  function buildGraph() {
    audioCtx = new (window.AudioContext || window.webkitAudioContext)();

    masterGain = audioCtx.createGain();
    masterGain.gain.value = 0; // will ramp up
    masterGain.connect(audioCtx.destination);

    // Effects chain
    return { masterGain };
  }

  function connectEffects(source) {
    let chain = source;
    if (filterOn && filterFreq < 20000) {
      const filter = audioCtx.createBiquadFilter();
      filter.type = 'lowpass';
      filter.frequency.value = filterFreq;
      filter.Q.value = filterRes;
      chain.connect(filter);
      chain = filter;
    }
    if (delayOn && delayTime > 0) {
      const delay = audioCtx.createDelay(2);
      delay.delayTime.value = delayTime;
      const feedback = audioCtx.createGain();
      feedback.gain.value = delayFeedback;
      const dryGain = audioCtx.createGain();
      dryGain.gain.value = 0.7;
      const wetGain = audioCtx.createGain();
      wetGain.gain.value = 0.3;
      chain.connect(dryGain);
      chain.connect(delay);
      delay.connect(wetGain);
      delay.connect(feedback);
      feedback.connect(delay);
      dryGain.connect(masterGain);
      wetGain.connect(masterGain);
      return;
    }
    chain.connect(masterGain);
  }

  // === FIXED: start with smooth ramp ===
  function startTone() {
    if (audioCtx) {
      audioCtx.close();
      audioCtx = null;
    }

    buildGraph();

    oscillator = audioCtx.createOscillator();
    oscillator.type = waveform;
    oscillator.frequency.value = frequency;
    connectEffects(oscillator);

    oscillator.start();

    // Smooth attack to avoid click
    const now = audioCtx.currentTime;
    masterGain.gain.setValueAtTime(0, now);
    masterGain.gain.linearRampToValueAtTime(volume / 100, now + 0.01);

    isPlaying = true;

    if (pulseMode) startPulse();
  }

  let oscillator = null;
  let currentOscillators = [];
  let pulseNodes = [];

  // === FIXED: recreate oscillator on waveform change ===
  function updateOscillator() {
    if (!audioCtx || !isPlaying) return;

    // Stop & disconnect old oscillator
    if (oscillator) {
      try { oscillator.stop(); } catch(e) {}
      oscillator.disconnect();
    }

    // Stop all additive oscillators
    for (const o of currentOscillators) {
      try { o.osc.stop(); } catch(e) {}
    }
    currentOscillators = [];
    // Disconnect all pulse scheduling nodes
    for (const n of pulseNodes) {
      try { n.disconnect(); } catch(e) {}
    }
    pulseNodes = [];

    if (activeTab === 'tone' || activeTab === 'effects') {
      // Single oscillator
      oscillator = audioCtx.createOscillator();
      oscillator.type = waveform;
      oscillator.frequency.value = frequency;
      connectEffects(oscillator);
      oscillator.start();
    } else if (activeTab === 'presets') {
      oscillator = buildPresetOscillator();
    } else if (activeTab === 'samples' && selectedSampleIndex >= 0) {
      oscillator = buildSamplePlayer();
    }

    if (pulseMode) startPulse();
  }

  function updateFrequency() {
    if (oscillator && isPlaying) {
      const now = audioCtx.currentTime;
      oscillator.frequency.setTargetAtTime(frequency, now, 0.02);
    }
  }

  function stopTone() {
    if (!audioCtx) return;

    // Smooth release to avoid click
    const now = audioCtx.currentTime;
    if (masterGain) {
      masterGain.gain.setValueAtTime(masterGain.gain.value, now);
      masterGain.gain.linearRampToValueAtTime(0, now + 0.01);
    }

    // Schedule cleanup after ramp completes
    setTimeout(() => {
      isPlaying = false;
      if (oscillator) {
        try { oscillator.stop(); } catch(e) {}
        oscillator.disconnect();
        oscillator = null;
      }
      for (const o of currentOscillators) {
        try { o.osc.stop(); } catch(e) {}
      }
      currentOscillators = [];
      for (const n of pulseNodes) {
        try { n.disconnect(); } catch(e) {}
      }
      pulseNodes = [];
      if (masterGain) {
        masterGain.disconnect();
        masterGain = null;
      }
      if (audioCtx) {
        audioCtx.close();
        audioCtx = null;
      }
    }, 20);
  }

  // === FIXED: AudioParam scheduling instead of setTimeout ===
  function startPulse() {
    if (!audioCtx || !masterGain || !isPlaying) return;

    const period = 1.0 / pulseRate;
    const onTime = period * (dutyCycle / 100);

    const now = audioCtx.currentTime;

    function schedulePulses(startTime) {
      if (!isPlaying || !audioCtx) return;
      const gain = masterGain.gain;
      gain.setValueAtTime(volume / 100, startTime);
      gain.setValueAtTime(0, startTime + onTime);

      // Schedule next pulse on the audio timeline
      const nextStart = startTime + period;
      // Use a small timeout to schedule ahead
      const ahead = (nextStart - audioCtx.currentTime) * 1000;
      if (ahead > 0 && ahead < 5000) {
        setTimeout(() => schedulePulses(nextStart), ahead);
      }
    }

    schedulePulses(now);
  }

  // === PRESETS: Algorithmic sound library ===
  const presets = [
    { id: 'sine', label: 'Pure Sine', desc: 'Clean fundamental only', icon: '∿' },
    { id: 'organ', label: 'Tonewheel Organ', desc: 'Classic Hammond organ', icon: '🎹', harmonics: [
      { ratio: 1, gain: 1 },
      { ratio: 2, gain: 0.5 },
      { ratio: 3, gain: 0.3 },
      { ratio: 4, gain: 0.15 },
      { ratio: 5, gain: 0.1 },
      { ratio: 6, gain: 0.05 }
    ]},
    { id: 'warmpad', label: 'Warm Pad', desc: 'Soft detuned sawtooth layers', icon: '🧣', harmonics: [
      { ratio: 1, gain: 1 },
      { ratio: 1.01, gain: 0.8, type: 'sawtooth' },
      { ratio: 2, gain: 0.4 },
      { ratio: 2.01, gain: 0.3, type: 'sawtooth' },
      { ratio: 3, gain: 0.2 }
    ]},
    { id: 'bass', label: 'Sub Bass', desc: 'Deep low-end thump', icon: '🔊', harmonics: [
      { ratio: 0.5, gain: 0.8, type: 'sine' },
      { ratio: 1, gain: 1, type: 'triangle' },
      { ratio: 2, gain: 0.3 },
      { ratio: 3, gain: 0.1 }
    ]},
    { id: 'bell', label: 'Bell Tone', desc: 'Metallic inharmonic partials', icon: '🔔', harmonics: [
      { ratio: 1, gain: 1 },
      { ratio: 2.76, gain: 0.5 },
      { ratio: 5.4, gain: 0.3 },
      { ratio: 7.1, gain: 0.15 },
      { ratio: 9.7, gain: 0.08 }
    ]},
    { id: 'pluck', label: 'Pluck', desc: 'Quick decaying string pluck', icon: '🎸', harmonics: [
      { ratio: 1, gain: 1, decay: 0.1 },
      { ratio: 2, gain: 0.5, decay: 0.05 },
      { ratio: 3, gain: 0.25, decay: 0.03 },
      { ratio: 4, gain: 0.125, decay: 0.02 },
      { ratio: 5, gain: 0.06, decay: 0.01 }
    ]},
    { id: 'noise', label: 'White Noise', desc: 'Full spectrum noise (wind/static)', icon: '🌬️', isNoise: true }
  ];

  let selectedPreset = presets[0];

  function buildPresetOscillator() {
    if (selectedPreset.isNoise) {
      return buildNoiseSource();
    }

    const harmonics = selectedPreset.harmonics || [{ ratio: 1, gain: 1 }];
    let mixGain = audioCtx.createGain();
    mixGain.gain.value = 1;

    currentOscillators = [];

    for (const h of harmonics) {
      const osc = audioCtx.createOscillator();
      osc.type = h.type || 'sine';
      osc.frequency.value = frequency * h.ratio;
      const gainNode = audioCtx.createGain();
      gainNode.gain.value = h.gain;

      // Apply decay envelope for pluck-like sounds
      if (h.decay) {
        const now = audioCtx.currentTime;
        gainNode.gain.setValueAtTime(h.gain, now);
        gainNode.gain.exponentialRampToValueAtTime(0.001, now + h.decay);
      }

      osc.connect(gainNode);
      gainNode.connect(mixGain);
      osc.start();
      currentOscillators.push({ osc, gain: gainNode, h });
    }

    connectEffects(mixGain);
    return mixGain;
  }

  function buildNoiseSource() {
    const bufferSize = audioCtx.sampleRate * 0.5;
    const buffer = audioCtx.createBuffer(1, bufferSize, audioCtx.sampleRate);
    const data = buffer.getChannelData(0);
    for (let i = 0; i < bufferSize; i++) {
      data[i] = Math.random() * 2 - 1;
    }
    const source = audioCtx.createBufferSource();
    source.buffer = buffer;
    source.loop = true;
    connectEffects(source);
    source.start();
    return source;
  }

  function applyPreset(preset) {
    selectedPreset = preset;
    if (isPlaying) {
      updateOscillator();
    }
  }

  // === CUSTOM SAMPLE UPLOAD ===
  let selectedSampleIndex = -1;
  let dragOver = false;

  function onFileDrop(e) {
    e.preventDefault();
    dragOver = false;
    const files = e.dataTransfer ? e.dataTransfer.files : (e.target ? e.target.files : []);
    for (const file of files) {
      if (file.type.startsWith('audio/')) {
        loadSample(file);
      }
    }
  }

  function onDragOver(e) {
    e.preventDefault();
    dragOver = true;
  }
  function onDragLeave() {
    dragOver = false;
  }

  function loadSample(file) {
    const reader = new FileReader();
    reader.onload = (e) => {
      const arrayBuffer = e.target.result;
      const tempCtx = new (window.AudioContext || window.webkitAudioContext)();
      tempCtx.decodeAudioData(arrayBuffer, (buffer) => {
        customSamples = [...customSamples, { name: file.name, buffer, duration: buffer.duration.toFixed(1) }];
        if (!isPlaying) {
          selectedSampleIndex = customSamples.length - 1;
        }
        tempCtx.close();
      }, () => {
        alert('Could not decode audio file. Try WAV or MP3.');
        tempCtx.close();
      });
    };
    reader.readAsArrayBuffer(file);
  }

  function buildSamplePlayer() {
    if (selectedSampleIndex < 0 || selectedSampleIndex >= customSamples.length) return;
    const sample = customSamples[selectedSampleIndex];
    const source = audioCtx.createBufferSource();
    source.buffer = sample.buffer;
    source.loop = true;
    connectEffects(source);
    source.start();
    return source;
  }

  function selectSample(index) {
    selectedSampleIndex = index;
    if (isPlaying) {
      updateOscillator();
    }
  }

  function removeSample(index) {
    customSamples = customSamples.filter((_, i) => i !== index);
    if (selectedSampleIndex === index) {
      selectedSampleIndex = -1;
    } else if (selectedSampleIndex > index) {
      selectedSampleIndex--;
    }
    if (isPlaying) {
      if (customSamples.length === 0) {
        stopTone();
      } else {
        updateOscillator();
      }
    }
  }

  // === UI Actions ===
  function updateVolume() {
    if (masterGain && isPlaying && !pulseMode) {
      const now = audioCtx.currentTime;
      masterGain.gain.setTargetAtTime(volume / 100, now, 0.02);
    }
  }

  function togglePlay() {
    if (isPlaying) stopTone();
    else startTone();
  }

  function onPulseToggle() {
    if (isPlaying) {
      stopTone();
      setTimeout(() => startTone(), 25);
    }
  }

  function onWaveformChange() {
    if (isPlaying) updateOscillator();
  }

  function switchTab(tab) {
    activeTab = tab;
    if (isPlaying) {
      stopTone();
      setTimeout(() => startTone(), 25);
    }
  }

  // Rebuild effects chain
  function rebuildEffects() {
    if (isPlaying && audioCtx) {
      // Restart with new effects
      stopTone();
      setTimeout(() => startTone(), 25);
    }
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

  <!-- Tab Bar -->
  <div class="tab-bar">
    <button class="tab" class:active={activeTab === 'tone'} on:click={() => switchTab('tone')}>Tone</button>
    <button class="tab" class:active={activeTab === 'presets'} on:click={() => switchTab('presets')}>Sounds</button>
    <button class="tab" class:active={activeTab === 'samples'} on:click={() => switchTab('samples')}>Samples</button>
    <button class="tab" class:active={activeTab === 'effects'} on:click={() => switchTab('effects')}>Effects</button>
  </div>

  <!-- === TAB: Tone Generator === -->
  {#if activeTab === 'tone'}
    <div class="controls-panel">
      <!-- Frequency -->
      <div class="control-group">
        <span class="control-label">Frequency</span>
        <div class="freq-row">
          <input type="range" min="0" max="1" step="0.001" bind:value={sliderPos} on:input={onSliderInput} class="freq-slider" />
          <input type="number" bind:value={freqInput} on:change={onFreqInput} class="freq-input" min="20" max="20000" />
          <span class="unit">Hz</span>
        </div>
        <div class="freq-markers"><span>20</span><span>100</span><span>1k</span><span>10k</span><span>20k</span></div>
      </div>

      <!-- Volume -->
      <div class="control-group">
        <span class="control-label">Volume</span>
        <div class="slider-row">
          <input type="range" min="0" max="100" bind:value={volume} on:input={updateVolume} />
          <span class="value-badge">{volume}%</span>
        </div>
      </div>

      <!-- Waveform -->
      <div class="control-group">
        <span class="control-label">Waveform</span>
        <div class="waveform-selector">
          {#each ['sine', 'square', 'sawtooth', 'triangle'] as wf}
            <button class="wave-btn" class:active={waveform === wf} on:click={() => { waveform = wf; onWaveformChange(); }}>{wf}</button>
          {/each}
        </div>
      </div>

      <!-- Pulse Mode -->
      <div class="control-group">
        <label class="control-label toggle-label">
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

      <button class="play-btn" class:playing={isPlaying} on:click={togglePlay}>
        {isPlaying ? '■ STOP' : '▶ PLAY'}
      </button>
    </div>

  <!-- === TAB: Sound Library (Presets) === -->
  {:else if activeTab === 'presets'}
    <div class="controls-panel">
      <p class="tab-desc">Algorithmic sound presets — no samples needed</p>
      <div class="preset-grid">
        {#each presets as preset}
          <button
            class="preset-card"
            class:active={selectedPreset.id === preset.id}
            on:click={() => applyPreset(preset)}
          >
            <span class="preset-icon">{preset.icon}</span>
            <span class="preset-label">{preset.label}</span>
            <span class="preset-desc">{preset.desc}</span>
          </button>
        {/each}
      </div>

      {#if isPlaying}
        <button class="play-btn playing" on:click={togglePlay}>■ STOP</button>
      {:else}
        <button class="play-btn" on:click={togglePlay}>▶ PLAY</button>
      {/if}
    </div>

  <!-- === TAB: Custom Samples === -->
  {:else if activeTab === 'samples'}
    <div class="controls-panel">
      <p class="tab-desc">Upload your own WAV, MP3, OGG, or M4A samples</p>

      <div
        class="drop-zone"
        class:dragover={dragOver}
        on:drop={onFileDrop}
        on:dragover={onDragOver}
        on:dragleave={onDragLeave}
        on:click={() => document.getElementById('file-input').click()}
      >
        {#if dragOver}
          ✨ Drop to add!
        {:else}
          📁 Drop audio files here or click to browse
        {/if}
      </div>
      <input type="file" id="file-input" accept="audio/*" multiple on:change={onFileDrop} style="display:none" />

      {#if customSamples.length > 0}
        <div class="sample-list">
          {#each customSamples as sample, i}
            <button
              class="sample-item"
              class:active={selectedSampleIndex === i}
              on:click={() => selectSample(i)}
            >
              <span class="sample-name">{sample.name}</span>
              <span class="sample-duration">{sample.duration}s</span>
              <button class="remove-btn" on:click|stopPropagation={() => removeSample(i)}>✕</button>
            </button>
          {/each}
        </div>

        {#if isPlaying}
          <button class="play-btn playing" on:click={togglePlay}>■ STOP</button>
        {:else}
          <button class="play-btn" on:click={togglePlay}>▶ PLAY CURRENT</button>
        {/if}
      {:else}
        <p class="empty-state">No samples loaded yet. Drop some audio files above!</p>
      {/if}
    </div>

  <!-- === TAB: Effects === -->
  {:else if activeTab === 'effects'}
    <div class="controls-panel">
      <!-- Low-pass Filter -->
      <div class="control-group">
        <label class="control-label toggle-label">
          <input type="checkbox" bind:checked={filterOn} on:change={rebuildEffects} class="toggle-checkbox" />
          Low-Pass Filter
        </label>
        {#if filterOn}
          <div class="sub-control">
            <label>Cutoff Freq</label>
            <div class="slider-row">
              <input type="range" min={20} max={20000} bind:value={filterFreq} on:input={rebuildEffects} />
              <span class="value-badge">{filterFreq} Hz</span>
            </div>
          </div>
          <div class="sub-control">
            <label>Resonance</label>
            <div class="slider-row">
              <input type="range" min="0" max="20" step="0.1" bind:value={filterRes} on:input={rebuildEffects} />
              <span class="value-badge">{filterRes.toFixed(1)}</span>
            </div>
          </div>
        {/if}
      </div>

      <!-- Delay / Echo -->
      <div class="control-group">
        <label class="control-label toggle-label">
          <input type="checkbox" bind:checked={delayOn} on:change={rebuildEffects} class="toggle-checkbox" />
          Delay / Echo
        </label>
        {#if delayOn}
          <div class="sub-control">
            <label>Delay Time</label>
            <div class="slider-row">
              <input type="range" min="0" max="1.5" step="0.01" bind:value={delayTime} on:input={rebuildEffects} />
              <span class="value-badge">{delayTime.toFixed(2)}s</span>
            </div>
          </div>
          <div class="sub-control">
            <label>Feedback</label>
            <div class="slider-row">
              <input type="range" min="0" max="0.95" step="0.01" bind:value={delayFeedback} on:input={rebuildEffects} />
              <span class="value-badge">{delayFeedback.toFixed(2)}</span>
            </div>
          </div>
        {/if}
      </div>

      {#if isPlaying}
        <button class="play-btn playing" on:click={togglePlay}>■ STOP</button>
      {:else}
        <button class="play-btn" on:click={togglePlay}>▶ PLAY</button>
      {/if}
    </div>
  {/if}
</div>

<style>
  .app-container {
    max-width: 640px;
    margin: 0 auto;
    padding: 0 20px 40px;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  }

  .header {
    display: flex;
    align-items: center;
    justify-content: center;
    position: relative;
    padding: 16px 0;
    border-bottom: 1px solid var(--border-color);
    margin-bottom: 16px;
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
    position: absolute;
    right: 0;
    top: 50%;
    transform: translateY(-50%);
  }
  .theme-btn svg { width: 16px; height: 16px; flex-shrink: 0; display: block; }
  .theme-btn:hover { border-color: var(--accent); }

  .tab-bar {
    display: flex;
    gap: 4px;
    margin-bottom: 20px;
    background: var(--surface-1);
    border-radius: 10px;
    padding: 4px;
    border: 1px solid var(--border-color);
  }
  .tab {
    flex: 1;
    padding: 8px 6px;
    background: transparent;
    color: var(--text-secondary);
    border: none;
    border-radius: 8px;
    font-size: 0.8rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.15s;
  }
  .tab.active {
    background: var(--accent);
    color: white;
  }
  .tab:hover:not(.active) { color: var(--text-primary); }

  .controls-panel {
    display: flex;
    flex-direction: column;
    gap: 16px;
  }

  .tab-desc {
    font-size: 0.85rem;
    color: var(--text-secondary);
    margin: 0;
    text-align: center;
  }

  .control-group {
    background: var(--surface-1);
    border: 1px solid var(--border-color);
    border-radius: 12px;
    padding: 16px 20px;
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
  .toggle-label { margin-bottom: 0; cursor: pointer; }

  .freq-row { display: flex; align-items: center; gap: 12px; }
  .freq-slider { flex: 1; }
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
  .unit { font-size: 0.85rem; color: var(--text-secondary); font-weight: 500; }
  .freq-markers {
    display: flex;
    justify-content: space-between;
    margin-top: 6px;
    font-size: 0.65rem;
    color: var(--text-secondary);
    padding: 0 2px;
  }

  .slider-row { display: flex; align-items: center; gap: 12px; }
  .slider-row input[type="range"] { flex: 1; }
  :global(input[type="range"]) { accent-color: var(--accent); }
  .value-badge {
    font-size: 0.85rem;
    font-weight: 600;
    color: var(--accent);
    min-width: 48px;
    text-align: right;
    font-variant-numeric: tabular-nums;
  }

  .waveform-selector { display: flex; gap: 6px; flex-wrap: wrap; }
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
  .wave-btn:hover { border-color: var(--accent); }
  .wave-btn.active { background: var(--accent); color: white; border-color: var(--accent); }

  .toggle-checkbox {
    width: 18px;
    height: 18px;
    cursor: pointer;
    accent-color: var(--accent);
  }

  .pulse-controls {
    display: flex;
    flex-direction: column;
    gap: 12px;
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
    cursor: pointer;
    transition: all 0.2s;
  }
  .play-btn:hover { filter: brightness(1.1); }
  .play-btn.playing { background: #d32f2f; }
  .play-btn.playing:hover { background: #b71c1c; }

  /* Preset Grid */
  .preset-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 10px;
  }
  .preset-card {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 4px;
    padding: 16px 10px;
    background: var(--surface-1);
    border: 1px solid var(--border-color);
    border-radius: 12px;
    cursor: pointer;
    transition: all 0.15s;
    color: var(--text-primary);
  }
  .preset-card:hover { border-color: var(--accent); }
  .preset-card.active {
    background: var(--accent);
    color: white;
    border-color: var(--accent);
  }
  .preset-card.active .preset-desc { color: rgba(255,255,255,0.7); }
  .preset-icon { font-size: 1.5rem; }
  .preset-label { font-size: 0.85rem; font-weight: 600; }
  .preset-desc { font-size: 0.7rem; color: var(--text-secondary); }

  /* Drop Zone */
  .drop-zone {
    border: 2px dashed var(--border-color);
    border-radius: 12px;
    padding: 32px 20px;
    text-align: center;
    color: var(--text-secondary);
    font-size: 0.9rem;
    cursor: pointer;
    transition: all 0.15s;
  }
  .drop-zone:hover, .drop-zone.dragover {
    border-color: var(--accent);
    background: var(--surface-1);
    color: var(--accent);
  }

  /* Sample List */
  .sample-list {
    display: flex;
    flex-direction: column;
    gap: 6px;
  }
  .sample-item {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 10px 14px;
    background: var(--surface-1);
    border: 1px solid var(--border-color);
    border-radius: 8px;
    cursor: pointer;
    transition: all 0.15s;
    color: var(--text-primary);
  }
  .sample-item:hover { border-color: var(--accent); }
  .sample-item.active {
    background: var(--accent);
    color: white;
    border-color: var(--accent);
  }
  .sample-item.active .sample-duration { color: rgba(255,255,255,0.7); }
  .sample-name { flex: 1; font-size: 0.85rem; font-weight: 500; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
  .sample-duration { font-size: 0.75rem; color: var(--text-secondary); font-variant-numeric: tabular-nums; }
  .remove-btn {
    background: none;
    border: none;
    color: var(--text-secondary);
    cursor: pointer;
    font-size: 0.9rem;
    padding: 2px 6px;
    border-radius: 4px;
  }
  .remove-btn:hover { color: #d32f2f; background: rgba(211,47,47,0.1); }
  .sample-item.active .remove-btn { color: rgba(255,255,255,0.7); }
  .sample-item.active .remove-btn:hover { color: white; background: rgba(255,255,255,0.2); }

  .empty-state {
    font-size: 0.85rem;
    color: var(--text-secondary);
    text-align: center;
    padding: 20px;
  }

  @media (max-width: 480px) {
    .freq-input { width: 72px; }
    .wave-btn { min-width: 60px; font-size: 0.75rem; }
    .preset-grid { grid-template-columns: 1fr 1fr; }
  }
</style>