<script lang="ts">
  import { onMount } from "svelte";
  import ReactionTimeTest from "../components/ReactionTimeTest.svelte";

  type Phase = "record" | "review";

  let phase = $state<Phase>("record");

  // Recording
  let videoEl = $state<HTMLVideoElement | null>(null);
  let stream = $state<MediaStream | null>(null);
  let mediaRecorder = $state<MediaRecorder | null>(null);
  let chunks = $state<Blob[]>([]);
  let isRecording = $state(false);
  let recordingStartTime = $state(0);

  // Review
  let reviewVideoEl = $state<HTMLVideoElement | null>(null);
  let recordedUrl = $state("");
  let videoFileInput = $state<HTMLInputElement | null>(null);
  let duration = $state(0);
  let currentTime = $state(0);
  let fps = $state(30);
  let fpsRead = $state(false);
  let measuringFps = $state(false);
  let _measureStop = false;
  let frameDuration = $derived(1 / fps);
  let totalFrames = $derived(Math.floor(duration / frameDuration));
  let currentFrame = $derived(Math.round(currentTime / frameDuration));
  let isDragging = $state(false);
  let scrubberTrack = $state<HTMLDivElement | null>(null);
  let scrubTimeout: ReturnType<typeof setTimeout> | null = null;

  // Markers
  let startMark = $state<number | null>(null);
  let endMark = $state<number | null>(null);
  let timeDiff = $derived(
    startMark !== null && endMark !== null
      ? Math.abs(endMark - startMark)
      : null,
  );

  // Velocity
  let distance = $state("");
  let inputUnit = $state<"meters" | "feet">("meters");
  let outputUnit = $state<"meters" | "feet">("meters");

  // Load preferences from storage on mount
  onMount(() => {
    const savedInputUnit = localStorage.getItem("inputUnit") as "meters" | "feet" | null;
    const savedOutputUnit = localStorage.getItem("outputUnit") as "meters" | "feet" | null;
    const savedDistance = localStorage.getItem("lastDistance");

    if (savedInputUnit) inputUnit = savedInputUnit;
    if (savedOutputUnit) outputUnit = savedOutputUnit;
    if (savedDistance) distance = savedDistance;
  });

  // Watch for changes and save to localStorage
  $effect(() => {
    localStorage.setItem("inputUnit", inputUnit);
  });

  $effect(() => {
    localStorage.setItem("outputUnit", outputUnit);
  });

  $effect(() => {
    localStorage.setItem("lastDistance", distance);
  });
  let velocity = $derived(() => {
    if (
      timeDiff === null ||
      !distance ||
      parseFloat(distance) <= 0 ||
      timeDiff === 0
    )
      return null;
    let d = parseFloat(distance);
    // Convert to meters if input is feet
    if (inputUnit === "feet") {
      d = d * 0.3048;
    }
    const v = d / timeDiff;
    return v;
  });
  let velocityDisplay = $derived(() => {
    const v = velocity();
    if (v === null) return null;
    let displayV = v;
    // Convert to feet/s if output is feet
    if (outputUnit === "feet") {
      displayV = v / 0.3048;
    }
    return `${displayV.toFixed(2)} ${outputUnit === "meters" ? "m/s" : "ft/s"}`;
  });

  async function startCamera() {
    try {
      stream = await navigator.mediaDevices.getUserMedia({
        video: { facingMode: "environment", frameRate: { ideal: 60, min: 30 } },
        audio: false,
      });
      if (videoEl) {
        videoEl.srcObject = stream;
      }
      // Try to read reported frameRate from the camera track settings
      const [track] = stream.getVideoTracks();
      if (track) {
        const settings = track.getSettings();
        if (
          settings &&
          typeof settings.frameRate === "number" &&
          isFinite(settings.frameRate)
        ) {
          fps = Math.round(settings.frameRate);
          fpsRead = true;
        } else {
          // If not reported, start a short measurement from playback as a fallback
          startMeasureFpsFromPlayback();
        }
      } else {
        startMeasureFpsFromPlayback();
      }
    } catch {
      alert("Could not access camera. Please grant permission.");
    }
  }

  function startMeasureFpsFromPlayback() {
    if (!videoEl) return;
    // Don't start if already measuring or already read
    if (measuringFps || fpsRead) return;
    if (typeof (videoEl as any).requestVideoFrameCallback !== "function") return;

    measuringFps = true;
    _measureStop = false;

    let last = -1;
    let frames = 0;
    let acc = 0; // seconds

    const cb = (now: number) => {
      if (_measureStop) {
        measuringFps = false;
        return;
      }
      if (last !== -1) {
        const dt = (now - last) / 1000;
        if (dt > 0) {
          acc += dt;
          frames++;
        }
      }
      last = now;

      if (acc >= 0.5 || frames >= 30) {
        const est = frames / (acc || 1);
        if (isFinite(est) && est > 0) {
          fps = Math.max(1, Math.round(est));
          fpsRead = true;
        }
        measuringFps = false;
        return;
      }

      (videoEl as any).requestVideoFrameCallback(cb);
    };

    try {
      (videoEl as any).requestVideoFrameCallback(cb);
    } catch {
      measuringFps = false;
    }
  }

  function stopMeasureFps() {
    _measureStop = true;
    measuringFps = false;
  }

  function toggleRecordingMark() {
    const now = performance.now() / 1000 - recordingStartTime;

    if (startMark === null) {
      // First click: mark start
      startMark = now;
    } else if (endMark === null) {
      // Second click: mark end
      endMark = now;
    } else {
      // Both marked: move to next pair (end becomes new start)
      startMark = endMark;
      endMark = now;
    }
  }

  function startRecording() {
    if (!stream) return;
    chunks = [];
    recordingStartTime = performance.now() / 1000;
    startMark = null;
    endMark = null;
    const options: MediaRecorderOptions = {};
    if (MediaRecorder.isTypeSupported("video/webm;codecs=vp9")) {
      options.mimeType = "video/webm;codecs=vp9";
    } else if (MediaRecorder.isTypeSupported("video/webm")) {
      options.mimeType = "video/webm";
    } else if (MediaRecorder.isTypeSupported("video/mp4")) {
      options.mimeType = "video/mp4";
    }
    mediaRecorder = new MediaRecorder(stream, options);
    mediaRecorder.ondataavailable = (e) => {
      if (e.data.size > 0) {
        chunks.push(e.data);
      }
    };
    mediaRecorder.onstop = () => {
      const blob = new Blob(chunks, {
        type: mediaRecorder?.mimeType || "video/webm",
      });
      recordedUrl = URL.createObjectURL(blob);
      phase = "review";
      stream?.getTracks().forEach((t) => t.stop());
      stream = null;
    };
    mediaRecorder.start();
    isRecording = true;
  }

  function stopRecording() {
    if (mediaRecorder && mediaRecorder.state !== "inactive") {
      mediaRecorder.stop();
      isRecording = false;
    }
  }

  function openVideoUpload() {
    if (videoFileInput) {
      videoFileInput.click();
    }
  }

  function onVideoFileSelected(e: Event) {
    const input = e.target as HTMLInputElement;
    const file = input.files?.[0];
    if (!file) return;

    // Verify it's a video file
    if (!file.type.startsWith("video/")) {
      alert("Please select a video file");
      return;
    }

    // Create object URL from the uploaded file
    if (recordedUrl) URL.revokeObjectURL(recordedUrl);
    recordedUrl = URL.createObjectURL(file);

    // Reset marks and move to review phase
    startMark = null;
    endMark = null;
    phase = "review";

    // Clear the input so the same file can be selected again if desired
    input.value = "";
  }

  function onReviewVideoLoaded() {
    if (!reviewVideoEl) return;
    duration = reviewVideoEl.duration;
    if (isNaN(duration) || !isFinite(duration)) {
      duration = 0;
      reviewVideoEl.currentTime = 1e10;
      setTimeout(() => {
        if (reviewVideoEl) {
          duration = reviewVideoEl.duration;
          reviewVideoEl.currentTime = 0;
        }
      }, 200);
    }
    reviewVideoEl.pause();
  }

  function seekToFrame(frame: number) {
    if (!reviewVideoEl) return;
    const t = Math.min(Math.max(frame * frameDuration, 0), duration);
    reviewVideoEl.currentTime = t;
    currentTime = t;
  }

  function stepFrame(delta: number) {
    seekToFrame(currentFrame + delta);
  }

  function scrubFromEvent(e: MouseEvent | Touch) {
    if (!scrubberTrack) return;
    const rect = scrubberTrack.getBoundingClientRect();
    const x = Math.max(0, Math.min(e.clientX - rect.left, rect.width));
    const ratio = x / rect.width;
    const frame = Math.round(ratio * totalFrames);

    // Clear pending timeout
    if (scrubTimeout) clearTimeout(scrubTimeout);

    // Debounce seek to 16ms (~60fps)
    scrubTimeout = setTimeout(() => {
      seekToFrame(frame);
      scrubTimeout = null;
    }, 16);
  }

  function onScrubStart(e: MouseEvent) {
    isDragging = true;
    scrubFromEvent(e);
  }

  function onScrubMove(e: MouseEvent) {
    if (!isDragging) return;
    scrubFromEvent(e);
  }

  function onScrubEnd() {
    isDragging = false;
    // Flush any pending scrub
    if (scrubTimeout) {
      clearTimeout(scrubTimeout);
      scrubTimeout = null;
    }
  }

  function onTouchScrubStart(e: TouchEvent) {
    isDragging = true;
    scrubFromEvent(e.touches[0]);
  }

  function onTouchScrubMove(e: TouchEvent) {
    if (!isDragging) return;
    e.preventDefault();
    scrubFromEvent(e.touches[0]);
  }

  function onTouchScrubEnd() {
    isDragging = false;
    // Flush any pending scrub
    if (scrubTimeout) {
      clearTimeout(scrubTimeout);
      scrubTimeout = null;
    }
  }

  function setStartMark() {
    startMark = currentTime;
    if (endMark !== null && endMark < startMark) {
      endMark = null;
    }
  }

  function setEndMark() {
    endMark = currentTime;
    if (startMark !== null && startMark > endMark) {
      startMark = null;
    }
  }

  function clearMarks() {
    startMark = null;
    endMark = null;
  }

  function goToStart() {
    if (startMark !== null && reviewVideoEl) {
      reviewVideoEl.currentTime = startMark;
      currentTime = startMark;
    }
  }

  function goToEnd() {
    if (endMark !== null && reviewVideoEl) {
      reviewVideoEl.currentTime = endMark;
      currentTime = endMark;
    }
  }

  function resetAll() {
    if (recordedUrl) URL.revokeObjectURL(recordedUrl);
    recordedUrl = "";
    duration = 0;
    currentTime = 0;
    startMark = null;
    endMark = null;
    distance = "";
    phase = "record";
    stopMeasureFps();
    // restart camera and measurement
    startCamera();
  }

  function formatTime(t: number): string {
    const mins = Math.floor(t / 60);
    const secs = t % 60;
    return `${mins}:${secs.toFixed(3).padStart(6, "0")}`;
  }

  onMount(() => {
    startCamera();

    const handleMouseMove = (e: MouseEvent) => onScrubMove(e);
    const handleMouseUp = () => onScrubEnd();
    window.addEventListener("mousemove", handleMouseMove);
    window.addEventListener("mouseup", handleMouseUp);

    return () => {
      window.removeEventListener("mousemove", handleMouseMove);
      window.removeEventListener("mouseup", handleMouseUp);
      if (scrubTimeout) clearTimeout(scrubTimeout);
      stopMeasureFps();
      stream?.getTracks().forEach((t) => t.stop());
      if (recordedUrl) URL.revokeObjectURL(recordedUrl);
    };
  });
</script>

<div class="app">
  <header>
    <h1>SHOT VELOCITY TIMER</h1>
  </header>

  {#if fpsRead}
    <div class="fps-badge">FPS: {fps}</div>
  {/if}

  <div class="top-bar">
    <h1>Shot Velocity Timer</h1>
    <ReactionTimeTest />
  </div>

  {#if phase === "record"}
    <section class="record-section">
      <div class="video-container">
        <video bind:this={videoEl} autoplay playsinline muted></video>
        {#if isRecording}
          <div class="rec-badge">● REC</div>
        {/if}
      </div>
      <div class="record-controls">
        {#if !isRecording}
          <div class="record-buttons">
            <button class="btn btn-record" onclick={startRecording}>
              ⬤ START RECORDING
            </button>
            <button class="btn btn-upload" onclick={openVideoUpload}>
              ↑ UPLOAD VIDEO
            </button>
          </div>
        {:else}
          <div class="recording-buttons">
            <button
              class="btn btn-mark"
              class:btn-mark-start={startMark === null}
              class:btn-mark-end={startMark !== null && endMark === null}
              class:btn-mark-next={startMark !== null && endMark !== null}
              onclick={toggleRecordingMark}
              title={startMark === null
                ? "Mark start"
                : endMark === null
                  ? "Mark end"
                  : "Mark new end (old end becomes start)"}
            >
              {#if startMark === null}
                ⬤ START
              {:else if endMark === null}
                ⬤ END
              {:else}
                ⬤ NEW END
              {/if}
            </button>
            <button class="btn btn-stop" onclick={stopRecording}>
              ■ STOP
            </button>
          </div>
        {/if}
      </div>

      <!-- Hidden video file input -->
      <input
        bind:this={videoFileInput}
        type="file"
        accept="video/*"
        onchange={onVideoFileSelected}
        style="display: none"
      />
    </section>
  {:else}
    <section class="review-section">
      <div class="video-container review-video-container">
        <video
          bind:this={reviewVideoEl}
          src={recordedUrl}
          onloadedmetadata={onReviewVideoLoaded}
          ontimeupdate={() => {
            if (reviewVideoEl && !isDragging)
              currentTime = reviewVideoEl.currentTime;
          }}
          playsinline
          preload="auto"
        ></video>
        <div class="frame-badge">
          Frame {currentFrame} / {totalFrames}
        </div>
      </div>

      <!-- Scrubber -->
      <div class="scrubber-section">
        <div class="time-display">
          <span>{formatTime(currentTime)}</span>
          <span>{formatTime(duration)}</span>
        </div>
        <div
          class="scrubber-track"
          bind:this={scrubberTrack}
          onmousedown={onScrubStart}
          ontouchstart={onTouchScrubStart}
          ontouchmove={onTouchScrubMove}
          ontouchend={onTouchScrubEnd}
          role="slider"
          aria-valuemin={0}
          aria-valuemax={totalFrames}
          aria-valuenow={currentFrame}
          tabindex="0"
        >
          {#if startMark !== null}
            <div
              class="marker marker-start"
              style="left: {(startMark / (duration || 1)) * 100}%"
            ></div>
          {/if}
          {#if endMark !== null}
            <div
              class="marker marker-end"
              style="left: {(endMark / (duration || 1)) * 100}%"
            ></div>
          {/if}
          {#if startMark !== null && endMark !== null}
            <div
              class="marker-range"
              style="left: {(Math.min(startMark, endMark) / (duration || 1)) *
                100}%; width: {(Math.abs(endMark - startMark) /
                (duration || 1)) *
                100}%"
            ></div>
          {/if}
          <div
            class="scrubber-fill"
            style="width: {(currentTime / (duration || 1)) * 100}%"
          ></div>
          <div
            class="scrubber-thumb"
            style="left: {(currentTime / (duration || 1)) * 100}%"
          ></div>
        </div>
      </div>

      <!-- Frame controls -->
      <div class="frame-controls">
        <button
          class="btn btn-frame"
          onclick={() => stepFrame(-1)}
          title="Back 1 frame"
        >
          ◀ -1
        </button>
        <button
          class="btn btn-frame"
          onclick={() => stepFrame(1)}
          title="Forward 1 frame"
        >
          +1 ▶
        </button>
      </div>

      <!-- Mark controls -->
      <div class="mark-controls">
        <button class="btn btn-start-mark" onclick={setStartMark}>
          SET START
        </button>
        <button class="btn btn-end-mark" onclick={setEndMark}> SET END </button>
        <button class="btn btn-clear" onclick={clearMarks}> CLEAR </button>
      </div>

      <!-- Marker info -->
      {#if startMark !== null || endMark !== null}
        <div class="marker-info">
          {#if startMark !== null}
            <button class="marker-chip chip-start" onclick={goToStart}>
              START: {formatTime(startMark)}
            </button>
          {/if}
          {#if endMark !== null}
            <button class="marker-chip chip-end" onclick={goToEnd}>
              END: {formatTime(endMark)}
            </button>
          {/if}
          {#if timeDiff !== null}
            <div class="time-diff-box">
              Δt = {timeDiff.toFixed(4)}s
            </div>
          {/if}
        </div>
      {/if}

      <!-- Velocity Calculator -->
      <div class="velocity-section">
        <h2>
          VELOCITY {#if timeDiff !== null}• Δt = {timeDiff.toFixed(4)}s{/if}
        </h2>
        <div class="velocity-row">
          <div class="input-group">
            <label for="distance-input">Distance</label>
            <input
              id="distance-input"
              type="number"
              step="any"
              min="0"
              placeholder="0.00"
              bind:value={distance}
            />
          </div>
          <div class="unit-toggle">
            <button
              class="toggle-btn"
              class:active={inputUnit === "meters"}
              onclick={() => (inputUnit = "meters")}
            >
              M
            </button>
            <button
              class="toggle-btn"
              class:active={inputUnit === "feet"}
              onclick={() => (inputUnit = "feet")}
            >
              FT
            </button>
          </div>
          <div class="unit-toggle">
            <button
              class="toggle-btn"
              class:active={outputUnit === "meters"}
              onclick={() => (outputUnit = "meters")}
            >
              M/S
            </button>
            <button
              class="toggle-btn"
              class:active={outputUnit === "feet"}
              onclick={() => (outputUnit = "feet")}
            >
              FT/S
            </button>
          </div>
        </div>
        {#if velocityDisplay() !== null}
          <div class="velocity-result">
            {velocityDisplay()}
          </div>
        {/if}
      </div>

      <!-- New recording -->
      <div class="new-recording">
        <button class="btn btn-new" onclick={resetAll}> NEW RECORDING </button>
      </div>
    </section>
  {/if}
</div>

<style>
  .app {
    max-width: 600px;
    margin: 0 auto;
    padding: 16px;
    min-height: 100vh;
  }

  header {
    text-align: center;
    margin-bottom: 20px;
  }

  header h1 {
    font-size: 1.6rem;
    font-weight: 900;
    letter-spacing: 2px;
    border: 3px solid #000;
    display: inline-block;
    padding: 8px 20px;
    background: rgba(255, 208, 116, 1);
    box-shadow: 4px 4px 0px #000;
  }

  /* Video */
  .video-container {
    position: relative;
    border: 3px solid #000;
    box-shadow: 8px 8px 0px #000;
    background: #000;
    overflow: hidden;
  }

  .video-container video {
    width: 100%;
    display: block;
  }

  .rec-badge {
    position: absolute;
    top: 12px;
    right: 12px;
    background: #ff3333;
    color: #fff;
    font-weight: 700;
    font-size: 0.85rem;
    padding: 4px 12px;
    border: 2px solid #000;
    box-shadow: 2px 2px 0px #000;
    animation: pulse 1s infinite;
  }

  @keyframes pulse {
    0%,
    100% {
      opacity: 1;
    }
    50% {
      opacity: 0.5;
    }
  }

  .frame-badge {
    position: absolute;
    top: 12px;
    right: 12px;
    background: rgba(0, 0, 0, 0.8);
    color: #fff;
    font-weight: 700;
    font-size: 0.8rem;
    padding: 4px 10px;
    border: 2px solid #000;
  }

  /* Record controls */
  .record-controls {
    margin-top: 16px;
    display: flex;
    justify-content: center;
  }

  .record-buttons {
    display: flex;
    flex-direction: column;
    gap: 8px;
    justify-content: center;
    align-items: stretch;
    width: 100%;
    max-width: 300px;
  }

  .recording-buttons {
  	display: flex;
  	flex-direction: column;
  	gap: 8px;
  	justify-content: center;
  	align-items: stretch;
  }

  /* Buttons */
  .btn {
    font-family: "DM Sans", sans-serif;
    font-weight: 700;
    font-size: 0.9rem;
    padding: 10px 20px;
    border: 3px solid #000;
    cursor: pointer;
    box-shadow: 2px 2px 0px #000;
    transition: all 0.15s ease;
    background: #fff;
    color: #000;
    letter-spacing: 1px;
  }

  .btn:hover {
    transform: translate(-1px, -1px);
    box-shadow: 3px 3px 0px #000;
  }

  .btn:active {
    transform: translate(1px, 1px);
    box-shadow: 0px 0px 0px #000;
  }

  .btn-record {
    background: rgba(255, 208, 116, 1);
    font-size: 1.1rem;
    padding: 14px 32px;
  }

  .btn-upload {
    background: rgba(176, 135, 255, 1);
    font-size: 1rem;
    padding: 12px 28px;
  }

  .btn-stop {
  	background: #ff3333;
  	color: #fff;
  	font-size: 1.1rem;
  	padding: 14px 32px;
  	width: 100%;
  }

  .btn-mark {
  	font-size: 0.95rem;
  	padding: 12px 16px;
  	width: 100%;
  }

  .btn-mark-start {
    background: rgba(23, 241, 209, 1);
  }

  .btn-mark-end {
    background: rgba(255, 208, 116, 1);
  }

  .btn-mark-next {
    background: rgba(176, 135, 255, 1);
  }

  /* Scrubber */
  .scrubber-section {
    margin-top: 16px;
  }

  .time-display {
    display: flex;
    justify-content: space-between;
    font-weight: 700;
    font-size: 0.85rem;
    margin-bottom: 6px;
    font-variant-numeric: tabular-nums;
  }

  .scrubber-track {
    position: relative;
    height: 40px;
    background: #e0e0e0;
    border: 3px solid #000;
    box-shadow: 2px 2px 0px #000;
    cursor: pointer;
    touch-action: none;
    user-select: none;
  }

  .scrubber-fill {
    position: absolute;
    top: 0;
    left: 0;
    height: 100%;
    background: rgba(176, 135, 255, 0.4);
    pointer-events: none;
  }

  .scrubber-thumb {
    position: absolute;
    top: -4px;
    width: 6px;
    height: calc(100% + 8px);
    background: #000;
    transform: translateX(-50%);
    pointer-events: none;
  }

  .marker {
    position: absolute;
    top: 0;
    width: 4px;
    height: 100%;
    transform: translateX(-50%);
    pointer-events: none;
    z-index: 2;
  }

  .marker-start {
    background: rgba(23, 241, 209, 1);
    box-shadow: 0 0 0 2px #000;
  }

  .marker-end {
    background: rgba(255, 208, 116, 1);
    box-shadow: 0 0 0 2px #000;
  }

  .marker-range {
    position: absolute;
    top: 0;
    height: 100%;
    background: rgba(23, 241, 209, 0.2);
    border-top: 3px dashed rgba(23, 241, 209, 0.6);
    border-bottom: 3px dashed rgba(23, 241, 209, 0.6);
    pointer-events: none;
    z-index: 1;
  }

  /* Frame controls */
  .frame-controls {
    display: flex;
    gap: 8px;
    margin-top: 12px;
    justify-content: center;
  }

  .btn-frame {
    flex: 0 1 auto;
    padding: 8px 12px;
    font-size: 0.75rem;
    background: #fff;
  }

  /* Mark controls */
  .mark-controls {
    display: flex;
    gap: 8px;
    margin-top: 12px;
  }

  .btn-start-mark {
    flex: 1;
    background: rgba(23, 241, 209, 1);
  }

  .btn-end-mark {
    flex: 1;
    background: rgba(255, 208, 116, 1);
  }

  .btn-clear {
    background: #fff;
  }

  /* Marker info */
  .marker-info {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-top: 8px;
    align-items: center;
  }

  .marker-chip {
    font-family: "DM Sans", sans-serif;
    font-weight: 700;
    font-size: 0.7rem;
    padding: 4px 10px;
    border: 2px solid #000;
    box-shadow: 1px 1px 0px #000;
    cursor: pointer;
    transition: all 0.15s ease;
    font-variant-numeric: tabular-nums;
  }

  .marker-chip:hover {
    transform: translate(-1px, -1px);
    box-shadow: 2px 2px 0px #000;
  }

  .chip-start {
    background: rgba(23, 241, 209, 1);
  }

  .chip-end {
    background: rgba(255, 208, 116, 1);
  }

  .time-diff-box {
    font-weight: 900;
    font-size: 0.75rem;
    padding: 4px 10px;
    border: 2px solid #000;
    background: rgba(176, 135, 255, 1);
    box-shadow: 1px 1px 0px #000;
    font-variant-numeric: tabular-nums;
  }

  /* Velocity */
  .velocity-section {
    margin-top: 16px;
    padding: 12px;
    border: 3px solid #000;
    box-shadow: 6px 6px 0px #000;
    background: #fff;
  }

  .velocity-section h2 {
    font-size: 0.95rem;
    font-weight: 900;
    letter-spacing: 1px;
    margin-bottom: 8px;
    font-variant-numeric: tabular-nums;
  }

  .velocity-row {
    display: flex;
    gap: 6px;
    align-items: flex-end;
  }

  .input-group {
    flex: 1;
    min-width: 0;
  }

  .input-group label {
    display: block;
    font-weight: 700;
    font-size: 0.65rem;
    margin-bottom: 2px;
    letter-spacing: 0.5px;
    text-transform: uppercase;
  }

  .input-group input {
    width: 100%;
    font-family: "DM Sans", sans-serif;
    font-weight: 700;
    font-size: 0.9rem;
    padding: 6px 8px;
    border: 2px solid #000;
    box-shadow: 1px 1px 0px #000;
    outline: none;
    background: #fff;
  }

  .input-group input:focus {
    box-shadow: 2px 2px 0px #000;
    transform: translate(-1px, -1px);
  }

  .unit-toggle {
    display: flex;
    border: 2px solid #000;
    box-shadow: 1px 1px 0px #000;
    overflow: hidden;
    flex-shrink: 0;
  }

  .toggle-btn {
    font-family: "DM Sans", sans-serif;
    font-weight: 700;
    font-size: 0.6rem;
    padding: 5px 8px;
    border: none;
    border-right: 1px solid #000;
    cursor: pointer;
    background: #e0e0e0;
    color: #000;
    transition: background 0.15s;
    letter-spacing: 0.5px;
    white-space: nowrap;
  }

  .toggle-btn:last-child {
    border-right: none;
  }

  .toggle-btn.active {
    background: rgba(255, 208, 116, 1);
  }

  .velocity-result {
    margin-top: 8px;
    font-weight: 900;
    font-size: 1.4rem;
    padding: 8px 12px;
    border: 3px solid #000;
    background: rgba(23, 241, 209, 1);
    box-shadow: 3px 3px 0px #000;
    text-align: center;
    letter-spacing: 0.5px;
    font-variant-numeric: tabular-nums;
  }

  /* New recording */
  .new-recording {
    margin-top: 20px;
    text-align: center;
    padding-bottom: 32px;
  }

  .btn-new {
    background: #fff;
    font-size: 1rem;
    padding: 12px 28px;
  }

  /* Top bar */
  .top-bar {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 20px;
    gap: 16px;
  }

  .top-bar h1 {
    font-size: 1.2rem;
    margin: 0;
    flex: 1;
  }

  .fps-badge {
    position: fixed;
    top: 12px;
    right: 12px;
    background: rgba(0, 0, 0, 0.8);
    color: #fff;
    font-weight: 700;
    font-size: 0.9rem;
    padding: 6px 10px;
    border: 2px solid #000;
    box-shadow: 2px 2px 0px #000;
    z-index: 9999;
  }
</style>
