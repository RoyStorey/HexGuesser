<script>
  import { onMount } from "svelte";

  const STATS_KEY = "daily-color-stats";
  const STATE_KEY = "daily-color-state";

  function getTodayKey() {
    return new Date().toISOString().split("T")[0];
  }

  function getDailyColor() {
    const today = getTodayKey();

    let seed = 0;
    for (let i = 0; i < today.length; i++) {
      seed = (seed * 31 + today.charCodeAt(i)) >>> 0;
    }

    function seededRandom() {
      seed = (seed * 9301 + 49297) % 233280;
      return seed / 233280;
    }

    const r = Math.floor(seededRandom() * 256);
    const g = Math.floor(seededRandom() * 256);
    const b = Math.floor(seededRandom() * 256);

    return `rgb(${r}, ${g}, ${b})`;
  }

  function rgbToHex(rgb) {
    const [r, g, b] = (rgb.match(/\d+/g) || []).map(Number);
    return `#${r.toString(16).padStart(2, "0")}${g
      .toString(16)
      .padStart(2, "0")}${b.toString(16).padStart(2, "0")}`.toUpperCase();
  }

  function normalizeHex(hex) {
    const cleaned = hex.replace("#", "").replace(/[^0-9a-fA-F]/g, "");
    if (cleaned.length < 6) {
      return "#" + cleaned.padEnd(6, "0").toUpperCase();
    }
    return "#" + cleaned.substring(0, 6).toUpperCase();
  }

  function hexToRgb(hex) {
    const cleaned = hex.replace("#", "");
    const r = parseInt(cleaned.substring(0, 2), 16);
    const g = parseInt(cleaned.substring(2, 4), 16);
    const b = parseInt(cleaned.substring(4, 6), 16);

    return {
      r: isNaN(r) ? 0 : r,
      g: isNaN(g) ? 0 : g,
      b: isNaN(b) ? 0 : b,
    };
  }

  function calculateColorSimilarity(color1, color2) {
    const rgb1 = hexToRgb(color1);
    const rgb2 = hexToRgb(color2);
    const maxDifference = 255 * 3;
    const difference =
      Math.abs(rgb1.r - rgb2.r) +
      Math.abs(rgb1.g - rgb2.g) +
      Math.abs(rgb1.b - rgb2.b);

    return Math.round(100 - (difference / maxDifference) * 100);
  }

  function getRgbDifferences(color1, color2) {
    const rgb1 = hexToRgb(color1);
    const rgb2 = hexToRgb(color2);

    return {
      r: Math.abs(rgb1.r - rgb2.r),
      g: Math.abs(rgb1.g - rgb2.g),
      b: Math.abs(rgb1.b - rgb2.b),
    };
  }

  function getDefaultStats() {
    return {
      gamesPlayed: 0,
      wins: 0,
      losses: 0,
      currentStreak: 0,
      bestStreak: 0,
      totalSimilarity: 0,
      totalGuesses: 0,
      averageSimilarity: 0,
      totalFirstGuessSimilarity: 0,
      firstGuessGames: 0,
      averageFirstGuessSimilarity: 0,
      totalThirdGuessSimilarity: 0,
      thirdGuessGames: 0,
      averageThirdGuessSimilarity: 0,
      lastPlayedDate: "",
    };
  }

  function loadStats() {
    if (typeof localStorage === "undefined") return getDefaultStats();

    try {
      const saved = localStorage.getItem(STATS_KEY);
      if (!saved) return getDefaultStats();

      const parsed = JSON.parse(saved);
      return {
        ...getDefaultStats(),
        ...parsed,
      };
    } catch {
      return getDefaultStats();
    }
  }

  function saveStats() {
    if (typeof localStorage === "undefined") return;
    localStorage.setItem(STATS_KEY, JSON.stringify(stats));
  }

  function loadDailyState() {
    if (typeof localStorage === "undefined") return;

    try {
      const saved = localStorage.getItem(STATE_KEY);
      if (!saved) return;

      const parsed = JSON.parse(saved);

      if (parsed.date !== todayKey) return;

      guesses = Array.isArray(parsed.guesses) ? parsed.guesses : [];
      feedback = parsed.feedback || "";
      guessCount = Number.isInteger(parsed.guessCount) ? parsed.guessCount : 0;
      displayedGuess = parsed.displayedGuess || "";
      gameOver = !!parsed.gameOver;
      gameWon = !!parsed.gameWon;
      showStats = !!parsed.showStats;
    } catch {
      // ignore invalid localStorage data
    }
  }

  function saveDailyState() {
    if (typeof localStorage === "undefined") return;

    const state = {
      date: todayKey,
      guesses,
      feedback,
      guessCount,
      displayedGuess,
      gameOver,
      gameWon,
      showStats,
    };

    localStorage.setItem(STATE_KEY, JSON.stringify(state));
  }

  function updateStatsForCompletedGame(didWin) {
    if (stats.lastPlayedDate === todayKey) return;

    const similaritySum = guesses.reduce(
      (sum, entry) => sum + entry.similarity,
      0,
    );

    stats.gamesPlayed += 1;

    if (didWin) {
      stats.wins += 1;
      stats.currentStreak += 1;
      if (stats.currentStreak > stats.bestStreak) {
        stats.bestStreak = stats.currentStreak;
      }
    } else {
      stats.losses += 1;
      stats.currentStreak = 0;
    }

    stats.totalSimilarity += similaritySum;
    stats.totalGuesses += guesses.length;
    stats.averageSimilarity =
      stats.totalGuesses > 0
        ? Math.round(stats.totalSimilarity / stats.totalGuesses)
        : 0;

    if (guesses.length >= 1) {
      stats.totalFirstGuessSimilarity += guesses[0].similarity;
      stats.firstGuessGames += 1;
      stats.averageFirstGuessSimilarity =
        stats.firstGuessGames > 0
          ? Math.round(stats.totalFirstGuessSimilarity / stats.firstGuessGames)
          : 0;
    }

    if (guesses.length >= 3) {
      stats.totalThirdGuessSimilarity += guesses[2].similarity;
      stats.thirdGuessGames += 1;
      stats.averageThirdGuessSimilarity =
        stats.thirdGuessGames > 0
          ? Math.round(stats.totalThirdGuessSimilarity / stats.thirdGuessGames)
          : 0;
    }

    stats.lastPlayedDate = todayKey;

    saveStats();
  }

  function finishGame(didWin) {
    gameWon = didWin;
    gameOver = true;
    updateStatsForCompletedGame(didWin);
    saveDailyState();
  }

  function handleGuessInput(event) {
    if (!(event.currentTarget instanceof HTMLInputElement)) return;
    if (guessCount >= 3 || gameOver) return;

    const value = event.currentTarget.value;
    const cleaned = value.replace(/[^0-9a-fA-F#]/g, "");

    if (cleaned.startsWith("#")) {
      guess =
        "#" + cleaned.slice(1).replace(/#/g, "").slice(0, 6).toUpperCase();
    } else {
      guess = cleaned.replace(/#/g, "").slice(0, 6).toUpperCase();
    }
  }

  function toggleStats() {
    showStats = !showStats;
  }

  let todayKey = getTodayKey();
  let randomColor = getDailyColor();
  let guess = "";
  let guesses = [];
  let feedback = "";
  let guessCount = 0;
  let displayedGuess = "";
  let gameOver = false;
  let gameWon = false;
  let showStats = false;
  let stats = getDefaultStats();

  $: winRate =
    stats.gamesPlayed > 0
      ? Math.round((stats.wins / stats.gamesPlayed) * 100)
      : 0;

  onMount(() => {
    stats = loadStats();
    loadDailyState();
  });

  $: saveDailyState();

  function handleGuess() {
    if (guess.replace("#", "").length < 6 || guessCount >= 3 || gameOver)
      return;

    const hexColor = rgbToHex(randomColor);
    const normalized = normalizeHex(guess);
    const similarity = calculateColorSimilarity(normalized, hexColor);

    guesses = [...guesses, { value: normalized, similarity }];
    displayedGuess = normalized;
    guessCount += 1;

    if (normalized.toLowerCase() === hexColor.toLowerCase()) {
      feedback = "Correct! Don't cheat. DARE to be different.";
      guess = "";
      finishGame(true);
      return;
    }

    if (guessCount === 3) {
      const diffs = getRgbDifferences(normalized, hexColor);
      feedback = `Final guess: ${normalized}. Today's color is ${hexColor}. Your final guess was off by R:${diffs.r}, G:${diffs.g}, B:${diffs.b}`;
      guess = "";
      finishGame(false);
      return;
    }

    feedback = `You guessed ${normalized}. (${similarity}% match) You have ${
      3 - guessCount
    } guess${3 - guessCount === 1 ? "" : "es"} left.`;

    guess = "";
  }
</script>

<div class="top-actions">
  <button type="button" class="stats-toggle" on:click={toggleStats}>
    {showStats ? "Hide Stats" : "View Stats"}
  </button>
</div>

{#if showStats}
  <div class="stats-panel">
    <h2>Stats</h2>
    <div class="stats-grid">
      <div class="stat-card">
        <span class="stat-label">Played</span>
        <strong>{stats.gamesPlayed}</strong>
      </div>
      <div class="stat-card">
        <span class="stat-label">Wins</span>
        <strong>{stats.wins}</strong>
      </div>
      <div class="stat-card">
        <span class="stat-label">Win Rate</span>
        <strong>{winRate}%</strong>
      </div>
      <div class="stat-card">
        <span class="stat-label">Current Streak</span>
        <strong>{stats.currentStreak}</strong>
      </div>
      <div class="stat-card">
        <span class="stat-label">Best Streak</span>
        <strong>{stats.bestStreak}</strong>
      </div>
      <div class="stat-card">
        <span class="stat-label">Avg Guess Match</span>
        <strong>{stats.averageSimilarity}%</strong>
      </div>
      <div class="stat-card">
        <span class="stat-label">Avg First Guess</span>
        <strong>{stats.averageFirstGuessSimilarity}%</strong>
      </div>
      <div class="stat-card">
        <span class="stat-label">Avg Third Guess</span>
        <strong>{stats.averageThirdGuessSimilarity}%</strong>
      </div>
    </div>
  </div>
{/if}

<section id="center">
  <div id="background" style="background-color: {randomColor}">
    <div
      class="color-display"
      style="background-color: {displayedGuess || 'transparent'}"
    ></div>

    <div class="game-section">
      <div class="guess-section">
        <p class="feedback">
          {feedback ||
            "Guess today's mystery color in hex. You only have 3 guesses."}
        </p>

        {#if !gameOver}
          <p class="guess-counter">Guess {guessCount + 1} of 3</p>
        {:else if gameWon}
          <p class="guess-counter">You solved today’s color</p>
        {:else}
          <p class="guess-counter">No guesses remaining</p>
        {/if}

        <input
          type="text"
          bind:value={guess}
          on:input={handleGuessInput}
          placeholder="#RRGGBB"
          maxlength="7"
          autocomplete="off"
          autocapitalize="characters"
          spellcheck="false"
          disabled={guessCount >= 3 || gameOver}
        />

        {#if !gameOver && guessCount < 3}
          <button
            type="button"
            on:click={handleGuess}
            disabled={guess.replace("#", "").length < 6}
          >
            Submit Guess
          </button>
        {/if}
      </div>

      <div class="previousGuesses">
        {#if guesses.length > 0}
          <h3>Today's guesses</h3>
          <ul class="guess-list">
            {#each guesses as guessEntry}
              <li class="guess">
                <strong>{guessEntry.value}</strong>
                <div
                  class="guessColorStrip"
                  style="background:{guessEntry.value}"
                ></div>
                <div>{guessEntry.similarity}% match</div>
              </li>
            {/each}
          </ul>
        {/if}
      </div>
    </div>
  </div>
</section>
