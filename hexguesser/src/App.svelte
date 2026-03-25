<script>
  function getDailyColor() {
    const today = new Date().toISOString().split('T')[0];

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

  /**
   * @param {string} rgb
   * @returns {string}
   */
  function rgbToHex(rgb) {
    const [r, g, b] = (rgb.match(/\d+/g) || []).map(Number);
    return `#${r.toString(16).padStart(2, '0')}${g.toString(16).padStart(2, '0')}${b.toString(16).padStart(2, '0')}`;
  }

  let randomColor = getDailyColor();
  let guess = '';
  let guesses = [];
  let feedback = '';
  let hasGuessed = false;
  let guessCount = 0;
  let displayedGuess = '';

  /**
   * @param {Event} event
   */
  function handleGuessInput(event) {
    // Only allow hex color format: #followed by 6 hex digits
    if (event.currentTarget instanceof HTMLInputElement) {
      const value = event.currentTarget.value;
      const hexPattern = /^#?[0-9a-fA-F]{0,6}$/;
      if (!hexPattern.test(value)) {
        // Revert to previous valid value if input is invalid
        guess = value.slice(0, -1);
      } else {
        guess = value;
      }
    }
  }

  /**
   * @param {string} hex
   * @returns {string}
   */
  function normalizeHex(hex) {
    const cleaned = hex.replace('#', '');
    if (cleaned.length < 6) {
      return '#' + cleaned.padEnd(6, '0');
    }
    return '#' + cleaned.substring(0, 6);
  }

  /**
   * @param {string} hex
   * @returns {{ r: number; g: number; b: number }}
   */
  function hexToRgb(hex) {
    const cleaned = hex.replace('#', '');
    const r = parseInt(cleaned.substring(0, 2), 16);
    const g = parseInt(cleaned.substring(2, 4), 16);
    const b = parseInt(cleaned.substring(4, 6), 16);
    return { r: isNaN(r) ? 0 : r, g: isNaN(g) ? 0 : g, b: isNaN(b) ? 0 : b };
  }

  /**
   * @param {string} color1
   * @param {string} color2
   * @returns {number}
   */
  function calculateColorSimilarity(color1, color2) {
    const rgb1 = hexToRgb(color1);
    const rgb2 = hexToRgb(color2);
    const maxDifference = 255 * 3; // Maximum possible difference
    const difference = Math.abs(rgb1.r - rgb2.r) + Math.abs(rgb1.g - rgb2.g) + Math.abs(rgb1.b - rgb2.b);
    return Math.round(100 - (difference / maxDifference) * 100);
  }

  /**
   * @param {string} color1
   * @param {string} color2
   * @returns {{ r: number; g: number; b: number }}
   */
  function getRgbDifferences(color1, color2) {
    const rgb1 = hexToRgb(color1);
    const rgb2 = hexToRgb(color2);
    return {
      r: Math.abs(rgb1.r - rgb2.r),
      g: Math.abs(rgb1.g - rgb2.g),
      b: Math.abs(rgb1.b - rgb2.b)
    };
  }

  function handleGuess() {
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
    hasGuessed = true;
  }
</script>


<section id="center">
  <div id="background" style="background-color: {randomColor}">
    <div class="color-display" style="background-color: {displayedGuess || 'transparent'}"></div>
    <div class="game-section">
    <div class="guess-section">
      <input type="text" bind:value={guess} on:input={handleGuessInput} placeholder="Guess the hex color" style="color:white; background: #aaaaaa"/>
      {#if guessCount < 3}
        <button type="button" on:click={handleGuess} disabled={guess.length < 6} style="background: #aaaaaa">Submit Guess</button>
      {/if}
      <p class="feedback">{feedback}</p>
    </div>
    <div class="previousGuesses">
        {#if guesses.length > 0}
          <h3>Today's guesses</h3>
          <ul>
            {#each guesses as guessEntry, index}
              <li>
                {index + 1}. <strong>{guessEntry.value}</strong> — {guessEntry.similarity}% match —  <div class="guessColorStrip" style="background:{normalizeHex(guessEntry.value)}"></div>
              </li>
            {/each}
          </ul>
        {/if}
      </div>
    </div>
  </div>
</section>