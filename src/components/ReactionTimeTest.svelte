<script lang="ts">
  import { onMount } from "svelte";

  interface ReactionTimeResult {
    times: number[];
    average: number;
    completed: boolean;
  }

  let isOpen = $state(false);
  let phase = $state<"idle" | "practice" | "practice-waiting" | "practice-testing" | "practice-result" | "waiting" | "testing" | "results">("idle");
  let round = $state(0);
  const totalRounds = 5;
  let reactionTimes = $state<number[]>([]);
  let currentStartTime = $state<number | null>(null);
  let testMessage = $state("");
  let stored = $state<ReactionTimeResult | null>(null);
  let practiceTime = $state<number | null>(null);

  onMount(() => {
    const saved = localStorage.getItem("reactionTimeTest");
    if (saved) {
      try {
        stored = JSON.parse(saved);
      } catch {
        stored = null;
      }
    }
  });

  function startTest() {
    isOpen = true;
    round = 0;
    reactionTimes = [];
    practiceTime = null;
    phase = "practice";
    testMessage = "Practice round first!";
  }

  function startPractice() {
    phase = "practice-waiting";
    testMessage = "Ready?";
    const delay = Math.random() * 3000 + 1000;
    setTimeout(() => {
      if (phase !== "practice-waiting") return;
      phase = "practice-testing";
      testMessage = "GO!";
      currentStartTime = performance.now();
    }, delay);
  }

  function recordPractice() {
    if (phase !== "practice-testing" || currentStartTime === null) return;
    practiceTime = (performance.now() - currentStartTime) / 1000;
    currentStartTime = null;
    phase = "practice-result";
    testMessage = `${(practiceTime * 1000).toFixed(0)}ms`;
  }

  function startRealTest() {
    round = 0;
    reactionTimes = [];
    phase = "waiting";
    nextRound();
  }

  function nextRound() {
    if (round >= totalRounds) {
      finishTest();
      return;
    }

    round++;
    testMessage = `Round ${round}/${totalRounds}`;
    phase = "waiting";

    const delay = Math.random() * 3000 + 1000; // 1-4 seconds
    setTimeout(() => {
      if (phase !== "waiting") return;
      phase = "testing";
      testMessage = "GO!";
      currentStartTime = performance.now();
    }, delay);
  }

  function recordReaction() {
    if (phase !== "testing" || currentStartTime === null) return;

    const reactionTime = (performance.now() - currentStartTime) / 1000;
    reactionTimes = [...reactionTimes, reactionTime];
    testMessage = `${(reactionTime * 1000).toFixed(0)}ms`;

    currentStartTime = null;
    phase = "waiting";

    setTimeout(() => {
      nextRound();
    }, 1500);
  }

  function finishTest() {
    const average =
      reactionTimes.reduce((a, b) => a + b, 0) / reactionTimes.length;
    const result: ReactionTimeResult = {
      times: reactionTimes,
      average,
      completed: true,
    };
    stored = result;
    localStorage.setItem("reactionTimeTest", JSON.stringify(result));
    phase = "results";
  }

  function resetTest() {
    localStorage.removeItem("reactionTimeTest");
    stored = null;
    isOpen = false;
    phase = "idle";
  }

  function closeModal() {
    isOpen = false;
  }
</script>

<div class="reaction-test">
  {#if stored?.completed}
    <button class="btn-reaction" onclick={startTest} title="Test again">
      ⚡ {(stored.average * 1000).toFixed(0)}ms
    </button>
  {:else}
    <button class="btn-reaction" onclick={startTest}>⚡ Reaction Test</button>
  {/if}
</div>

{#if isOpen}
  <div class="modal-overlay" role="presentation" onclick={closeModal} onkeydown={(e) => {
    if (e.key === "Escape") closeModal();
  }}>
    <div class="modal" role="dialog" tabindex={0} onclick={(e) => e.stopPropagation()} onkeydown={(e) => e.stopPropagation()}>
      {#if phase === "idle"}
        <button class="btn-tap-full" onclick={startTest}>
          Start?
        </button>
      {:else if phase === "practice"}
        <button class="btn-tap-full" onclick={startPractice}>
          Start Practice Round
        </button>
      {:else if phase === "practice-waiting" || phase === "practice-testing"}
        <button
          class="btn-tap-full"
          class:active={phase === "practice-testing"}
          onclick={recordPractice}
          onkeydown={(e) => {
            if (e.code === "Space") {
              e.preventDefault();
              recordPractice();
            }
          }}
        >
          {phase === "practice-testing" ? testMessage : "Click when Blue"}
        </button>
      {:else if phase === "practice-result"}
        <button class="btn-tap-full" onclick={startRealTest}>
          {(practiceTime && practiceTime * 1000).toFixed(0)}ms - Start Real Test
        </button>
      {:else if phase === "waiting" || phase === "testing"}
        <button
          class="btn-tap-full"
          class:active={phase === "testing"}
          onclick={recordReaction}
          onkeydown={(e) => {
            if (e.code === "Space") {
              e.preventDefault();
              recordReaction();
            }
          }}
        >
          {phase === "testing" ? testMessage : `Round ${round}/${totalRounds}`}
        </button>
      {:else if phase === "results"}
        <button class="btn-tap-full" onclick={startTest}>
          {((stored?.average ?? 0) * 1000).toFixed(0)}ms - Test Again
        </button>
      {/if}
    </div>
  </div>
{/if}

<style>
  .reaction-test {
    display: inline-block;
  }

  .btn-reaction {
    font-family: "DM Sans", sans-serif;
    font-weight: 900;
    font-size: 0.9rem;
    padding: 10px 16px;
    border: 3px solid #000;
    background: rgba(255, 100, 100, 1);
    box-shadow: 3px 3px 0px #000;
    cursor: pointer;
    transition: all 0.15s ease;
    letter-spacing: 0.5px;
  }

  .btn-reaction:hover {
    transform: translate(-2px, -2px);
    box-shadow: 5px 5px 0px #000;
  }

  .btn-reaction:active {
    transform: translate(0, 0);
    box-shadow: 2px 2px 0px #000;
  }

  .modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
  }

  .modal {
    background: #fff;
    border: 4px solid #000;
    box-shadow: 8px 8px 0px #000;
    padding: 0;
    max-width: 500px;
    width: 90%;
    border-radius: 0;
  }

  .btn-tap-full {
    width: 100%;
    padding: 60px 24px;
    font-size: 1.8rem;
    font-weight: 900;
    background: #d0d0d0;
    border: none;
    border-radius: 0;
    cursor: pointer;
    transition: all 0.1s ease;
    font-family: "DM Sans", sans-serif;
    letter-spacing: 0.5px;
    color: #888;
    line-height: 1.2;
    text-align: center;
    word-break: break-word;
  }

  .btn-tap-full:hover {
    background: #c0c0c0;
  }

  .btn-tap-full.active {
    background: rgba(23, 241, 209, 1);
    color: #000;
    animation: pulse 0.3s ease-in-out;
  }

  .btn-tap-full.active:hover {
    background: rgba(23, 241, 209, 0.9);
  }

  .btn-tap-full.active:active {
    background: rgba(23, 241, 209, 0.8);
  }

  @keyframes pulse {
    0% {
      transform: scale(1);
    }
    50% {
      transform: scale(1.02);
    }
    100% {
      transform: scale(1);
    }
  }
</style>
