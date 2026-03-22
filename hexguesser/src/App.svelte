<script>
  function generateRandomRgbColor() {
    const r = Math.floor(Math.random() * 256); // Random number between 0-255
    const g = Math.floor(Math.random() * 256);
    const b = Math.floor(Math.random() * 256);

    return `rgb(${r}, ${g}, ${b})`;
  }

  /**
   * @param {string} rgb
   * @returns {string}
   */
  function getComplementaryColor(rgb) {
    const [r, g, b] = (rgb.match(/\d+/g) || []).map(Number);
    if ([r, g, b].some((v) => Number.isNaN(v))) return '#000';
    return `rgb(${255 - r}, ${255 - g}, ${255 - b})`;
  }

  /**
   * @param {string} rgb
   * @returns {string}
   */
  function rgbToHex(rgb) {
    const [r, g, b] = (rgb.match(/\d+/g) || []).map(Number);
    return `#${r.toString(16).padStart(2, '0')}${g.toString(16).padStart(2, '0')}${b.toString(16).padStart(2, '0')}`;
  }

  let randomColor = generateRandomRgbColor();
  let guess = '';
  let feedback = '';
  let hasGuessed = false;

  function handleGenerateNewColor() {
    randomColor = generateRandomRgbColor();
    feedback = ''; // Reset feedback on new color
    guess = ''; // Clear the input field
    hasGuessed = false;
  }

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

  function handleGuess() {
    const hexColor = rgbToHex(randomColor);
    if (guess.toLowerCase() === hexColor.toLowerCase()) {
      feedback = 'Correct!';
    } else {
      const similarity = calculateColorSimilarity(guess, hexColor);
      feedback = `You guessed #${guess}. The color is ${hexColor} (${similarity}% match)`;
    }
    hasGuessed = true;
  }
</script>


<section id="center">
  <div id="background" style="background-color: {randomColor}">

    <div class="color-display" style="background-color: {hasGuessed ? normalizeHex(guess) : 'transparent'}"></div>
    <div class="guess-section">
    
      <input type="text" bind:value={guess} on:input={handleGuessInput} placeholder="Guess the hex color (e.g., #ff0000)" style="color:white; background: #aaaaaa"/>
      {#if !hasGuessed}
        <button type="button" on:click={handleGuess} disabled={guess.length < 6} style="background: #aaaaaa">Submit Guess</button>
      {/if}
      <p style="color: #aaaaaa">{feedback}</p>
      {#if hasGuessed}
        <button
          class="button"
          type="button"
          on:click={handleGenerateNewColor}
          style="background: #aaaaaa"
        >
          <p>Generate New Color</p>
        </button>
      {/if}
    </div>
  </div>
</section>