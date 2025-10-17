<script>
  import { onMount } from 'svelte';
  
  // State management
  let balls = 0;
  let overs = 0;
  let showResetModal = false;
  
  
  // Undo slider state
  let isDragging = false;
  let holdTimer = null;
  let holdProgress = 0;
  let isHolding = false;
  let sliderElement = null;
  
  // Wake lock
  let wakeLock = null;
  
  // Audio
  let clickSound = null;
  
  // Load state from localStorage on mount
  onMount(() => {
    loadState();
    initializeWakeLock();
    initializeAudio();
    registerServiceWorker();
  });
  
  function loadState() {
    try {
      const savedState = localStorage.getItem('cricket-counter-state');
      if (savedState) {
        const { balls: savedBalls, overs: savedOvers } = JSON.parse(savedState);
        balls = savedBalls || 0;
        overs = savedOvers || 0;
      }
    } catch (error) {
      console.error('Failed to load state:', error);
    }
  }
  
  function saveState() {
    try {
      localStorage.setItem('cricket-counter-state', JSON.stringify({ balls, overs }));
    } catch (error) {
      console.error('Failed to save state:', error);
    }
  }
  
  function initializeWakeLock() {
    if ('wakeLock' in navigator) {
      requestWakeLock();
    }
    
    // Reacquire wake lock when page becomes visible
    document.addEventListener('visibilitychange', () => {
      if (document.visibilityState === 'visible' && !wakeLock) {
        requestWakeLock();
      }
    });
  }
  
  async function requestWakeLock() {
    try {
      wakeLock = await navigator.wakeLock.request('screen');
      wakeLock.addEventListener('release', () => {
        wakeLock = null;
      });
    } catch (error) {
      console.error('Wake lock failed:', error);
    }
  }
  
  function initializeAudio() {
    // Create a simple click sound using Web Audio API
    const audioContext = new (window.AudioContext || window.webkitAudioContext)();
    
    function playClickSound() {
      const oscillator = audioContext.createOscillator();
      const gainNode = audioContext.createGain();
      
      oscillator.connect(gainNode);
      gainNode.connect(audioContext.destination);
      
      oscillator.frequency.setValueAtTime(800, audioContext.currentTime);
      oscillator.type = 'sine';
      
      gainNode.gain.setValueAtTime(0.3, audioContext.currentTime);
      gainNode.gain.exponentialRampToValueAtTime(0.01, audioContext.currentTime + 0.1);
      
      oscillator.start(audioContext.currentTime);
      oscillator.stop(audioContext.currentTime + 0.1);
    }
    
    clickSound = playClickSound;
  }
  
  function registerServiceWorker() {
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.register('/sw.js')
        .then((registration) => {
          console.log('Service Worker registered successfully:', registration);
        })
        .catch((error) => {
          console.log('Service Worker registration failed:', error);
        });
    }
  }
  
  function vibrate(pattern) {
    if ('vibrate' in navigator) {
      try {
        navigator.vibrate(pattern);
        console.log('Vibration triggered:', pattern);
      } catch (error) {
        console.log('Vibration failed:', error);
      }
    } else {
      console.log('Vibration not supported on this device/browser');
    }
  }
  
  function incrementBall() {
    balls++;
    
    // Play sound and vibrate
    if (clickSound) {
      clickSound();
    }
    vibrate(50); // Single vibration
    
    // Check if over is complete (6th ball)
    if (balls >= 6) {
      balls = 0;
      overs++;
      vibrate([100, 50, 100, 50, 100]); // Triple vibration pattern
    }
    
    saveState();
  }
  
  function undoBall() {
    if (balls > 0) {
      balls--;
      saveState();
    }
  }
  
  function resetCurrentOver() {
    balls = 0;
    saveState();
    showResetModal = false;
  }
  
  function resetGame() {
    balls = 0;
    overs = 0;
    saveState();
    showResetModal = false;
  }
  
  // Simple drag handlers
  let dragStartX = 0;
  let currentX = 0;
  
  function handleMouseDown(event) {
    event.preventDefault();
    isDragging = true;
    dragStartX = event.clientX;
    currentX = event.clientX;
    
    document.addEventListener('mousemove', handleMouseMove);
    document.addEventListener('mouseup', handleMouseUp);
  }
  
  function handleTouchStart(event) {
    event.preventDefault();
    isDragging = true;
    dragStartX = event.touches[0].clientX;
    currentX = event.touches[0].clientX;
    
    document.addEventListener('touchmove', handleTouchMove);
    document.addEventListener('touchend', handleTouchEnd);
  }
  
  function handleMouseMove(event) {
    if (!isDragging) return;
    event.preventDefault();
    currentX = event.clientX;
    updateDragProgress();
  }
  
  function handleTouchMove(event) {
    if (!isDragging) return;
    event.preventDefault();
    currentX = event.touches[0].clientX;
    updateDragProgress();
  }
  
  function handleMouseUp(event) {
    if (!isDragging) return;
    isDragging = false;
    resetSlider();
    
    document.removeEventListener('mousemove', handleMouseMove);
    document.removeEventListener('mouseup', handleMouseUp);
  }
  
  function handleTouchEnd(event) {
    if (!isDragging) return;
    isDragging = false;
    resetSlider();
    
    document.removeEventListener('touchmove', handleTouchMove);
    document.removeEventListener('touchend', handleTouchEnd);
  }
  
  function updateDragProgress() {
    const deltaX = dragStartX - currentX; // Positive when dragging left
    const sliderWidth = sliderElement ? sliderElement.offsetWidth : 300;
    const progress = Math.max(0, Math.min(1, deltaX / (sliderWidth * 0.6)));
    
    // If dragged far enough to the left, start hold timer
    if (progress > 0.6 && !isHolding) {
      isHolding = true;
      holdProgress = 0;
      
      holdTimer = setInterval(() => {
        holdProgress += 20;
        if (holdProgress >= 200) { // 2 seconds
          undoBall();
          resetSlider();
        }
      }, 20);
    } else if (progress <= 0.6 && isHolding) {
      // Reset if not held far enough
      isHolding = false;
      holdProgress = 0;
      if (holdTimer) {
        clearInterval(holdTimer);
        holdTimer = null;
      }
    }
  }
  
  function resetSlider() {
    isDragging = false;
    isHolding = false;
    holdProgress = 0;
    
    if (holdTimer) {
      clearInterval(holdTimer);
      holdTimer = null;
    }
  }
  
  // Calculate slider position - handle starts on right (6px) and follows finger when dragged
  $: sliderPosition = isDragging ? Math.max(2, Math.min(288, 3 + (dragStartX - currentX))) : 3;
  $: holdProgressPercent = isHolding ? (holdProgress / 200) * 100 : 0;
</script>

<svelte:head>
  <title>Cricket Counter</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <meta name="theme-color" content={showResetModal ? '#000000' : '#ffffff'}>
  <meta name="apple-mobile-web-app-status-bar-style" content={showResetModal ? 'black-translucent' : 'default'}>
  <meta name="apple-mobile-web-app-capable" content="yes">
  
  <!-- PWA Meta Tags -->
  <link rel="manifest" href="/manifest.json">
  <meta name="apple-mobile-web-app-title" content="Cricket Counter">
  <meta name="application-name" content="Cricket Counter">
  <meta name="msapplication-TileColor" content="#ffffff">
  <meta name="msapplication-tap-highlight" content="no">
  
  <!-- Icons -->
  <link rel="icon" type="image/png" href="/ball.png">
  <link rel="apple-touch-icon" href="/ball.png">
</svelte:head>

<div id="app">
  <!-- Header -->
  <header class="header">
    <button class="reset-button" on:click={() => showResetModal = true}>
      RESET
    </button>
    
    <div class="counters">
      <div class="counter">
        <div class="counter-label">OVERS</div>
        <div class="counter-value">{overs}</div>
      </div>
      <div class="counter">
        <div class="counter-label">BALLS</div>
        <div class="counter-value">{balls}</div>
      </div>
    </div>
  </header>
  
  <!-- Cricket Ball Button -->
  <div class="cricket-ball-container">
    <button 
      class="cricket-ball" 
      on:click={incrementBall}
      aria-label="Increment ball count"
    >
      <img src="/ball.svg" alt="Cricket Ball" class="cricket-ball-svg" />
    </button>
  </div>
  
  <!-- Undo Slider -->
  <div class="undo-container">
    <div 
      class="undo-slider"
      bind:this={sliderElement}
      role="slider"
      aria-label="Undo ball count slider"
      aria-valuenow={isHolding ? holdProgress : 0}
      aria-valuemin="0"
      aria-valuemax="200"
      tabindex="0"
    >
      <div class="undo-track" class:active={isHolding}></div>
      <div class="undo-label">REMOVE DELIVERY</div>
      <div 
        class="undo-handle" 
        class:dragging={isDragging}
        role="button"
        tabindex="0"
        aria-label="Drag to remove last ball"
        style="right: {sliderPosition}px;"
        on:mousedown={handleMouseDown}
        on:touchstart={handleTouchStart}
      >
        <div class="undo-handle-icon">‹</div>
        <div 
          class="undo-progress-ring" 
          class:active={isHolding}
          style="--progress: {holdProgressPercent}%"
        ></div>
      </div>
    </div>
  </div>
</div>

<!-- Reset Modal -->
{#if showResetModal}
  <div class="modal-overlay" role="dialog" aria-modal="true" tabindex="-1" on:click={() => showResetModal = false} on:keydown={(e) => e.key === 'Escape' && (showResetModal = false)}>
    <div class="modal" role="document" on:click|stopPropagation on:keydown|stopPropagation>
      <h2 class="modal-title">Reset Options</h2>
      <div class="modal-buttons">
        <button class="modal-button primary" on:click={resetCurrentOver}>
          Reset Current Over
        </button>
        <button class="modal-button primary" on:click={resetGame}>
          Reset Game
        </button>
        <button class="modal-button secondary" on:click={() => showResetModal = false}>
          Cancel
        </button>
      </div>
    </div>
  </div>
{/if}

<style>
  @import '../app.css';
</style>
