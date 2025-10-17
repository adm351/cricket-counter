<script>
  import { onMount } from 'svelte';
  
  // State management
  let balls = 0;
  let overs = 0;
  let showResetModal = false;
  
  // Undo slider state
  let isDragging = false;
  let dragStartX = 0;
  let currentX = 0;
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
  
  function vibrate(pattern) {
    if ('vibrate' in navigator) {
      navigator.vibrate(pattern);
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
  
  // Undo slider handlers
  function handleSliderStart(event) {
    event.preventDefault();
    event.stopPropagation();
    
    isDragging = true;
    const clientX = event.type === 'mousedown' ? event.clientX : event.touches[0].clientX;
    dragStartX = clientX;
    currentX = clientX;
    
    // Add global event listeners for better responsiveness
    document.addEventListener('mousemove', handleSliderMove, { passive: false });
    document.addEventListener('mouseup', handleSliderEnd, { passive: false });
    document.addEventListener('touchmove', handleSliderMove, { passive: false });
    document.addEventListener('touchend', handleSliderEnd, { passive: false });
  }
  
  function handleSliderMove(event) {
    if (!isDragging) return;
    
    event.preventDefault();
    event.stopPropagation();
    
    const clientX = event.type === 'mousemove' ? event.clientX : event.touches[0].clientX;
    const deltaX = dragStartX - clientX; // Positive when dragging left (undo direction)
    const sliderWidth = sliderElement ? sliderElement.offsetWidth : 300;
    const progress = Math.max(0, Math.min(1, deltaX / (sliderWidth * 0.7))); // More sensitive threshold
    
    currentX = clientX;
    
    // If dragged far enough to the left, start hold timer
    if (progress > 0.6 && !isHolding) {
      isHolding = true;
      holdProgress = 0;
      
      holdTimer = setInterval(() => {
        holdProgress += 20; // Increment by 20ms for smoother progress
        if (holdProgress >= 2000) { // 2 seconds
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
  
  function handleSliderEnd(event) {
    if (!isDragging) return;
    
    event.preventDefault();
    event.stopPropagation();
    
    isDragging = false;
    isHolding = false;
    holdProgress = 0;
    
    if (holdTimer) {
      clearInterval(holdTimer);
      holdTimer = null;
    }
    
    // Remove global event listeners
    document.removeEventListener('mousemove', handleSliderMove);
    document.removeEventListener('mouseup', handleSliderEnd);
    document.removeEventListener('touchmove', handleSliderMove);
    document.removeEventListener('touchend', handleSliderEnd);
    
    resetSlider();
  }
  
  function resetSlider() {
    currentX = dragStartX;
    isDragging = false;
    isHolding = false;
    holdProgress = 0;
  }
  
  // Calculate slider position - handle starts on right (6px) and follows finger when dragged
  $: sliderPosition = isDragging ? Math.max(2, Math.min(248, 6 - (currentX - dragStartX))) : 6;
  $: holdProgressPercent = isHolding ? (holdProgress / 2000) * 100 : 0;
</script>

<svelte:head>
  <title>Cricket Counter</title>
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
      aria-valuemax="2000"
      tabindex="0"
      on:mousedown={handleSliderStart}
      on:touchstart={handleSliderStart}
    >
      <div class="undo-track" class:active={isHolding}></div>
      <div class="undo-label">REMOVE</div>
      <div 
        class="undo-handle" 
        class:dragging={isDragging}
        style="right: {sliderPosition}px;"
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
