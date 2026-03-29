<script>
  function getDailyColor() {
    const today = new Date().toISOString().split("T")[0];

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

  function handleGuessInput(event) {
    if (!(event.currentTarget instanceof HTMLInputElement)) return;

    const value = event.currentTarget.value;
    const cleaned = value.replace(/[^0-9a-fA-F#]/g, "");

    if (cleaned.startsWith("#")) {
      guess =
        "#" + cleaned.slice(1).replace(/#/g, "").slice(0, 6).toUpperCase();
    } else {
      guess = cleaned.replace(/#/g, "").slice(0, 6).toUpperCase();
    }
  }

  let randomColor = getDailyColor();
  let guess = "";
  let guesses = [];
  let feedback = "";
  let guessCount = 0;
  let displayedGuess = "";

  function handleGuess() {
    if (guess.replace("#", "").length < 6 || guessCount >= 3) return;

    const hexColor = rgbToHex(randomColor);
    const normalized = normalizeHex(guess);
    const similarity = calculateColorSimilarity(normalized, hexColor);

    guesses = [...guesses, { value: normalized, similarity }];
    displayedGuess = normalized;

    if (normalized.toLowerCase() === hexColor.toLowerCase()) {
      feedback = "Correct! Don't cheat. DARE to be different.";
    } else if (guessCount === 2) {
      const diffs = getRgbDifferences(normalized, hexColor);
      feedback = `Final guess: ${normalized}. Today's color is ${hexColor}. Your final guess was off by R:${diffs.r}, G:${diffs.g}, B:${diffs.b}`;
    } else {
      feedback = `You guessed ${normalized}. (${similarity}% match)`;
    }

    guessCount += 1;
    guess = "";
  }
</script>

<section id="center">
  <div id="background" style="background-color: {randomColor}">
    <div
      class="color-display"
      style="background-color: {displayedGuess || 'transparent'}"
    ></div>

    <div class="game-section">
      <div class="guess-section">
        <p class="feedback">
          {feedback || "Guess today's mystery color in hex."}
        </p>

        <input
          type="text"
          bind:value={guess}
          on:input={handleGuessInput}
          placeholder="#RRGGBB"
          maxlength="7"
          autocomplete="off"
          autocapitalize="characters"
          spellcheck="false"
        />

        {#if guessCount < 3}
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
