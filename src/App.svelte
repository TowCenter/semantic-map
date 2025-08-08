<script>
  import { onMount } from 'svelte';
  import Papa from 'papaparse';
  import Scatterplot from './components/Scatterplot.svelte';
  import RangeSlider from './components/RangeSlider.svelte';

  let data = [], columns = [], domainColumn = "org", uniqueValues = [];
  let selectedValues = new Set(); 
  let opacity = 0.5, startDate = null, endDate = null;
  let filteredData = [], allDates = [];
  let startDateIndex = 0;
  let endDateIndex = 0;
  let isPlaying = false;
  let playInterval = null;
  let highlightedData = [];
  $: highlightedSet = new Set(highlightedData.map(d => d.id));


  let searchQuery = "";
  let showAnnotations = true;

  $: filteredData = data.map(d => ({
    ...d,
    isHighlighted: 
      (selectedValues.size > 0 && selectedValues.size < uniqueValues.length) && 
      selectedValues.has(d[domainColumn]) &&
      (!startDate || d.date >= startDate) &&
      (!endDate || d.date <= endDate)
  }));


  $: startPercent = allDates.length > 1 ? (startDateIndex / (allDates.length - 1)) * 100 : 0;
  $: endPercent = allDates.length > 1 ? (endDateIndex / (allDates.length - 1)) * 100 : 100;

  onMount(async () => {
    const response = await fetch('data.csv');
    const csvText = await response.text();
    parseCSV(csvText);

    return () => {
      if (playInterval) clearInterval(playInterval);
    };
  });

  function parseCSV(csvText) {
    const result = Papa.parse(csvText, { header: true });
    data = result.data.filter(d => d.x && d.y && d.date)
    .map((d, i) => ({ ...d, x: +d.x, y: +d.y, date: new Date(d.date), id: i }))


    columns = result.meta.fields || [];
    allDates = [...new Set(data.map(d => d.date.getTime()))]
      .sort((a, b) => a - b)
      .map(t => new Date(t));

    startDate = allDates[0];
    endDate = allDates[allDates.length - 1];
    startDateIndex = 0;
    endDateIndex = allDates.length - 1;

    if (domainColumn) {
      uniqueValues = [...new Set(data.map(d => d[domainColumn]).filter(Boolean))];
    }
  }

  function handleDomainChange(event) {
    domainColumn = event.target.value;
    uniqueValues = domainColumn ? [...new Set(data.map(d => d[domainColumn]).filter(Boolean))] : [];
    selectedValues = new Set();
    showAnnotations = false;
  }

  function handleSelectionChange(event) {
    const selectedOptions = [...event.target.selectedOptions].map(o => o.value);
    
    // Check if all values are selected
    const allSelected = selectedOptions.length === uniqueValues.length;
    
    if (allSelected || selectedOptions.length === 0) {
      // If all values are selected or none are selected, clear the selection
      selectedValues = new Set();
      highlightedData = [];
    } else {
      // Only update selection if a subset is chosen
      selectedValues = new Set(selectedOptions);
      highlightedData = data.filter(d =>
        selectedValues.has(d[domainColumn]) &&
        (!startDate || d.date >= startDate) &&
        (!endDate || d.date <= endDate)
      );
    }
    
    showAnnotations = false;
  }

  function updateSelectedDates(start, end, fromIndices = false) {
    if (fromIndices) {
      // If we're updating from indices, convert them to actual dates
      startDate = allDates[start] || allDates[0];
      endDate = allDates[end] || allDates[allDates.length - 1];
    } else {
      // We're updating from actual date objects
      startDate = start;
      endDate = end;
    }
    showAnnotations = false;
  }

  function updateDateIndices() {
    // Find closest date indices based on the current startDate and endDate
    if (startDate) {
      const timestamp = startDate.getTime();
      startDateIndex = Math.max(0, allDates.findIndex(d => d.getTime() >= timestamp));
    } else {
      startDateIndex = 0;
    }

    if (endDate) {
      const timestamp = endDate.getTime();
      const exactIndex = allDates.findIndex(d => d.getTime() >= timestamp);
      endDateIndex = exactIndex >= 0 ? 
        exactIndex : 
        (allDates.length > 0 ? allDates.length - 1 : 0);
    } else {
      endDateIndex = allDates.length - 1;
    }

    // Ensure indices are in valid range
    if (startDateIndex > endDateIndex) startDateIndex = endDateIndex;
    if (endDateIndex < startDateIndex) endDateIndex = startDateIndex;
  }

  function handleDateChange(e, type) {
    if (e.detail !== undefined) {
      // Handle slider events
      if (type === 'start') {
        startDateIndex = e.detail;
        if (startDateIndex > endDateIndex) startDateIndex = endDateIndex;
      } else {
        endDateIndex = e.detail;
        if (endDateIndex < startDateIndex) endDateIndex = startDateIndex;
      }
      // Update dates based on the indices
      updateSelectedDates(startDateIndex, endDateIndex, true);
    } else {
      // Handle date input events
      const newDate = e.target.value ? new Date(e.target.value) : null;
      
      if (type === 'start') {
        updateSelectedDates(newDate, endDate);
      } else {
        updateSelectedDates(startDate, newDate);
      }
      
      // Update indices based on the new dates
      updateDateIndices();
    }
  }

  function formatDateInput(date) {
    return date ? date.toISOString().split('T')[0] : '';
  }

  function handleFileUpload(event) {
    const file = event.target.files[0];
    if (file) {
      const reader = new FileReader();
      showAnnotations = false;
      reader.onload = (e) => parseCSV(e.target.result);
      reader.readAsText(file);
    }
  }

  function shiftDateRange(days) {
    const rangeDuration = endDateIndex - startDateIndex;
    let newStartIndex = startDateIndex + days;
    let newEndIndex = endDateIndex + days;

    if (newStartIndex < 0) {
      newStartIndex = 0;
      newEndIndex = rangeDuration;
    }

    if (newEndIndex >= allDates.length) {
      newEndIndex = allDates.length - 1;
      newStartIndex = Math.max(0, newEndIndex - rangeDuration);
    }

    startDateIndex = newStartIndex;
    endDateIndex = newEndIndex;
    updateSelectedDates(startDateIndex, endDateIndex, true);
  }

  function togglePlayPause() {
    if (isPlaying) {
      clearInterval(playInterval);
      isPlaying = false;
    } else {
      isPlaying = true;
      showAnnotations = false;
      playInterval = setInterval(() => {
        if (endDateIndex >= allDates.length - 1) {
          clearInterval(playInterval);
          isPlaying = false;
          return;
        }
        shiftDateRange(1);
      }, 500);
    }
  }

  function handleSearch(event) {
    searchQuery = event.target.value;
    showAnnotations = false;
  }
  
  function handleOpacityChange() {
    showAnnotations = false;
  }
</script>

<!-- App Layout -->
<div class="container">
  <div class="title-section">
    <h1 class="title">Semantic Map [DEMO]</h1>
    <p class="subtitle">
      Each dot represents an article. When the dots are closer together, the articles are similar in meaning.
    </p>
  </div>

  <div class="content">
    <!-- Filters Panel -->
    <div class="filter-panel">

      <div class="nerd-box">
        <details close>
          <summary>➕ What is a Semantic Map?</summary>
          <div class="nerd-box-content">
            <p>
              Semantic maps are tools that allow users to visually explore how different topics and ideas are connected. 
              By analyzing the relationships between articles, these maps can reveal patterns and clusters in the data.
            </p>
            <p>
              Newsrooms can use semantic maps to:
            </p>
            <ul>
              <li>Audit their coverage by identifying themes that may be underrepresented or overlooked.</li>
              <li>Track how their editorial focus evolves over time.</li>
              <li>Discover which topics are being covered by other outlets, offering opportunities to adjust their editorial focus or collaborate.</li>
            </ul>
          </div>
        </details>
      </div>

      <label for="file-upload">📁 Upload CSV:</label>
      <input id="file-upload" type="file" accept=".csv" on:change={handleFileUpload} />
      
      <label for="search-input">🔍 Search Text (supports regex):</label>
      <input id="search-input" type="text" placeholder="Search or regex pattern..." on:input={handleSearch} />

      {#if columns.length}
        <label for="domain-column">🎨 Color by Column:</label>
        <select id="domain-column" on:change={handleDomainChange} bind:value={domainColumn}>
          <option value="" disabled>Select column</option>
          {#each columns as column}
            <option value={column}>{column}</option>
          {/each}
        </select>
      {/if}

      {#if uniqueValues.length}
        <label for="value-select">✨ Highlight Values:</label>
        <select id="value-select" multiple size="5" class="multi-select" on:change={handleSelectionChange}>
          {#each uniqueValues as value}
            <option value={value}>{value}</option>
          {/each}
        </select>
      {/if}

      <label for="opacity-slider">💡 Adjust Opacity:</label>
      <input id="opacity-slider" type="range" min="0.01" max="1" step="0.1" bind:value={opacity} on:input={handleOpacityChange} />

      <div class="date-controls">
        <label for="start-date">📅 Date Range:</label>
        <input id="start-date" type="date" value={formatDateInput(startDate)} on:change={(e) => handleDateChange(e, 'start')} />
        <input id="end-date" type="date" value={formatDateInput(endDate)} on:change={(e) => handleDateChange(e, 'end')} />

        <RangeSlider 
          min={0} 
          max={allDates.length - 1} 
          bind:startValue={startDateIndex} 
          bind:endValue={endDateIndex}
          on:startChange={(e) => handleDateChange(e, 'start')}
          on:endChange={(e) => handleDateChange(e, 'end')}
        />

        <div class="date-range-labels">
          <span>{formatDateInput(startDate)}</span>
          <span>{formatDateInput(endDate)}</span>
        </div>

        <div class="date-navigation">
          <button on:click={() => shiftDateRange(-7)}>-1W</button>
          <button on:click={() => shiftDateRange(-1)}>-1D</button>
          <button class="play-button" on:click={togglePlayPause}>
            {#if isPlaying}
              ❚❚
            {:else}
              ▶
            {/if}
          </button>
          <button on:click={() => shiftDateRange(1)}>+1D</button>
          <button on:click={() => shiftDateRange(7)}>+1W</button>
        </div>
      </div>
    </div>

    <div class="scatterplot-container">
      {#if filteredData.length}
        <Scatterplot 
          data={filteredData} 
          {domainColumn} 
          {selectedValues} 
          {opacity} 
          {searchQuery}
          {showAnnotations}
          {highlightedData}
          {startDate}    
          {endDate}      
        />
      {:else}
        <p>Loading data...</p>
      {/if}
    </div>
  </div>
</div>

<style>
  .container {
    padding: 1rem;
    display: flex;
    flex-direction: column;
    gap: 1rem;
    font-family: 'Inter', sans-serif;
  }

  .title-section {
    text-align: center;
  }

  .title {
    font-size: 2rem;
    font-weight: 700;
    margin-bottom: 0.5rem;
  }

  .subtitle {
    font-size: 1.1rem;
    color: #666;
    width: 60%;
    margin: 0 auto;
    line-height: 1.5;
  }

  .content {
    display: flex;
    gap: 1.5rem;
    align-items: flex-start; /* Align items at the top */
  }

  .filter-panel {
    background: #f9fafb;
    /* border-right: #969696 1px solid; */
    padding: 1.5rem;
    width: 300px; 
    display: flex;
    flex-direction: column;
    gap: 1rem;
    max-height: 100vh; 
    position: sticky; 
    overflow: auto;
    top: 1rem; 
  }

  .filter-panel label {
    font-weight: 600;
    font-size: 0.9rem;
  }

  .filter-panel input,
  .filter-panel select {
    padding: 0.5rem;
    font-size: 0.9rem;
    border: 1px solid #ccc;
    border-radius: 5px;
    width: 100%; 
    box-sizing: border-box; 
  }

  .multi-select {
    min-height: 150px;
    overflow-y: auto;
    width: 100%; 
  }

  .filter-panel input:focus,
  .filter-panel select:focus {
    outline: none;
    border-color: #4c8bf5;
  }

  .scatterplot-container {
    flex-grow: 1;
    display: flex;
    justify-content: center;
    align-items: center;
    background: #fff;
    border-radius: 10px;
    min-height: 500px;
    padding: 1rem;
    /* box-shadow: inset 0 0 10px rgba(0, 0, 0, 0.05); */
  }

  .date-controls label {
    display: block; 
    margin-bottom: 0.5rem; 
    font-weight: 600; 
  }

  .date-controls input[type="date"] {
    margin-bottom: 0.5rem;
  }

  .date-range-labels {
    display: flex;
    justify-content: space-between;
    font-size: 0.85rem;
    color: #444;
  }

  .date-navigation {
    display: flex;
    gap: 0.5rem;
    justify-content: space-between;
  }

  .date-navigation button {
    flex: 1;
    padding: 0.4rem;
    font-size: 0.85rem;
    background: #f0f0f0;
    border: 1px solid #ccc;
    border-radius: 4px;
    cursor: pointer;
  }

  .play-button {
    background: #4c8bf5;
    color: white;
    font-weight: bold;
  }

  .nerd-box {
    background: #eef5ff;
    border: 1px solid #4c8bf5;
    padding: 1rem;
    border-radius: 8px;
    font-size: 0.9rem;
    line-height: 1.6;
    margin-bottom: 1rem;
  }

  .nerd-box summary {
    /* toggle */
    
    font-weight: bold;
    cursor: pointer;
    list-style: none;
    margin-bottom: 0.5rem;
  }

  .nerd-box-content p {
    margin-bottom: 1rem;
    color: #444;
  }

  .nerd-box-content ul {
    list-style-type: disc; /* Add bullet points */
    padding-left: 1.5rem; /* Indent the list */
    margin-bottom: 1rem; /* Add spacing below the list */
  }

  .nerd-box-content li {
    margin-bottom: 0.5rem; /* Add spacing between list items */
    color: #444;
  }
</style>