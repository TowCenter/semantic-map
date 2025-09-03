<script>
  import { onMount } from 'svelte';
  import Papa from 'papaparse';
  import Scatterplot from './components/Scatterplot.svelte';
  import RangeSlider from './components/RangeSlider.svelte';
  import { labelForDomain, descriptionForDomain } from './labels';

  const DATA_URL = import.meta.env.VITE_DATA_URL || 'data.csv';
  $: isDefaultRemote = /^https?:\/\//i.test(DATA_URL);
  const DATE_CAP_STR = '2025-07-01';
  const MAX_DATE_CAP = new Date(DATE_CAP_STR);
  
  let data = [], columns = [], domainColumn = "org", uniqueValues = [];
  let selectedValues = new Set(); 
  let opacity = 0.3, startDate = null, endDate = null;
  let filteredData = [], allDates = [];
  let startDateIndex = 0;
  let endDateIndex = 0;
  let isPlaying = false;
  let playInterval = null;
  let highlightedData = [];
  $: highlightedSet = new Set(highlightedData.map(d => d.id));


  let searchQuery = "";
  let showAnnotations = false;

  // Only allow org or cluster as color-by options
  $: allowedDomainColumns = columns.filter((c) => c === 'org' || c === 'cluster');
  $: {
    // Wait until columns are known before adjusting the selected domain
    if (!columns || columns.length === 0) {
      // do nothing until parsed
    } else if (allowedDomainColumns.length && !allowedDomainColumns.includes(domainColumn)) {
      domainColumn = allowedDomainColumns[0];
    } else if (allowedDomainColumns.length === 0) {
      domainColumn = '';
    }
  }

  // Keep unique values in sync as data or domain selection changes
  $: uniqueValues = domainColumn ? [...new Set(data.map(d => d[domainColumn]).filter(v => v !== undefined && v !== null && v !== ''))] : [];
  // Compute human-readable labels for UI (values remain raw for selection)
  $: labeledUniqueValues = uniqueValues.map(v => ({ value: v, label: labelForDomain(domainColumn, v, isDefaultRemote) }));

  // Loading/progress state
  let isLoading = false;
  let loadPhase = 'idle'; // 'downloading' | 'parsing' | 'idle'
  let loadBytes = 0;
  let loadTotal = 0;
  let loadProgress = 0; // 0-100 when total known

  function formatBytes(bytes) {
    if (!bytes || bytes <= 0) return '0 KB';
    const units = ['B', 'KB', 'MB', 'GB', 'TB'];
    let v = bytes;
    let i = 0;
    while (v >= 1024 && i < units.length - 1) {
      v /= 1024;
      i++;
    }
    const dp = i === 0 ? 0 : 1;
    return `${v.toFixed(dp)} ${units[i]}`;
  }

  onMount(async () => {
    try {
      isLoading = true;
      loadPhase = 'downloading';
      loadBytes = 0;
      loadTotal = 0;
      loadProgress = 0;

      const response = await fetch(DATA_URL, { mode: 'cors', cache: 'no-store' });
      if (!response.ok) throw new Error(`HTTP ${response.status}`);

      const len = response.headers.get('content-length');
      loadTotal = len ? parseInt(len, 10) : 0;

      let csvText = '';
      if (response.body && response.body.getReader) {
        const reader = response.body.getReader();
        const chunks = [];
        let received = 0;
        // Stream and accumulate while reporting progress
        while (true) {
          const { done, value } = await reader.read();
          if (done) break;
          chunks.push(value);
          received += value.byteLength;
          loadBytes = received;
          if (loadTotal) {
            loadProgress = Math.round((received / loadTotal) * 100);
          }
        }
        const full = new Uint8Array(received);
        let offset = 0;
        for (const c of chunks) {
          full.set(c, offset);
          offset += c.byteLength;
        }
        csvText = new TextDecoder().decode(full);
      } else {
        // Fallback without streaming/progress
        csvText = await response.text();
      }

      // Parse phase (indeterminate)
      loadPhase = 'parsing';
      await Promise.resolve(parseCSV(csvText));
      loadPhase = 'idle';
    } catch (err) {
      console.error('Failed to load CSV:', err);
    } finally {
      isLoading = false;
    }

    return () => {
      if (playInterval) clearInterval(playInterval);
    };
  });  

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

  // Highest index allowed under the cap date (last date <= MAX_DATE_CAP)
  $: maxAllowedIndex = (() => {
    if (!allDates || allDates.length === 0) return 0;
    const capTs = MAX_DATE_CAP.getTime();
    // find first index with date > cap, then step back one
    const firstAfter = allDates.findIndex(d => d.getTime() > capTs);
    return firstAfter === -1 ? allDates.length - 1 : Math.max(0, firstAfter - 1);
  })();



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
  // Clamp initial end date to cap so input isn't invalid
    if (endDate && endDate.getTime() > MAX_DATE_CAP.getTime()) {
      endDate = new Date(MAX_DATE_CAP);
    }
  startDateIndex = 0;
  // set to the last permissible index under the cap
  endDateIndex = maxAllowedIndex;
  // Recompute indices to reflect any clamping
  updateDateIndices();

    if (domainColumn) {
      uniqueValues = [...new Set(data.map(d => d[domainColumn]).filter(v => v !== undefined && v !== null && v !== ''))];
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

  // Keep highlightedData in sync is handled by selection handlers for multi-select

  function updateSelectedDates(start, end, fromIndices = false) {
    if (fromIndices) {
      // If we're updating from indices, convert them to actual dates
      startDate = allDates[start] || allDates[0];
      endDate = allDates[end] || allDates[allDates.length - 1];
      if (endDate && endDate.getTime() > MAX_DATE_CAP.getTime()) {
        endDate = new Date(MAX_DATE_CAP);
      }
    } else {
      // We're updating from actual date objects
      startDate = start;
      endDate = end ? new Date(Math.min(end.getTime(), MAX_DATE_CAP.getTime())) : end;
    }
    showAnnotations = false;
  // Keep indices in sync with possibly clamped dates
  updateDateIndices();
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
      const capTs = MAX_DATE_CAP.getTime();
      const timestamp = Math.min(endDate.getTime(), capTs);
      // choose the last index whose date <= timestamp (works with caps and inputs between dates)
      let ei = -1;
      for (let i = allDates.length - 1; i >= 0; i--) {
        if (allDates[i].getTime() <= timestamp) { ei = i; break; }
      }
      endDateIndex = ei >= 0 ? ei : 0;
    } else {
      endDateIndex = maxAllowedIndex;
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

  function resetFilters() {
    // Stop playback if running
    if (isPlaying && playInterval) {
      clearInterval(playInterval);
      isPlaying = false;
    }

    // Reset domain to default ('org' if available, otherwise first allowed)
    const defaultDomain = columns.includes('org') ? 'org' : (allowedDomainColumns[0] || '');
    domainColumn = defaultDomain;

    // Clear selection and highlights
    selectedValues = new Set();
    highlightedData = [];

    // Reset search and annotations
    searchQuery = '';
    showAnnotations = false;

    // Reset opacity to initial default
    opacity = 0.3;

    // Reset date range to full
    if (allDates.length) {
      startDateIndex = 0;
      endDateIndex = allDates.length - 1;
      updateSelectedDates(startDateIndex, endDateIndex, true);
    } else {
      startDate = null;
      endDate = null;
      startDateIndex = 0;
      endDateIndex = 0;
    }
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

      <div class="filter-actions">
        <button class="reset-btn" on:click={resetFilters}>Reset filters</button>
      </div>

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

      {#if allowedDomainColumns.length}
        <label for="domain-column">🎨 Color by Column:</label>
        <select id="domain-column" on:change={handleDomainChange} bind:value={domainColumn}>
          <option value="" disabled>Select column</option>
          {#each allowedDomainColumns as column}
            <option value={column}>{column}</option>
          {/each}
        </select>
      {/if}

      {#if uniqueValues.length}
        <label for="value-select">✨ Highlight Values (hold Shift/Cmd to select multiple):</label>
    <select id="value-select" multiple size="5" class="multi-select" on:change={handleSelectionChange}>
          {#each labeledUniqueValues as item}
      <option value={item.value} selected={selectedValues.has(item.value)}>{item.label}</option>
          {/each}
        </select>
      {/if}

      <label for="opacity-slider">💡 Adjust Opacity of Active Articles:</label>
      <input id="opacity-slider" type="range" min="0.01" max="1" step="0.1" bind:value={opacity} on:input={handleOpacityChange} />

      <div class="date-controls">
        <label for="start-date">📅 Date Range:</label>
  <input id="start-date" type="date" value={formatDateInput(startDate)} max="2025-07-01" on:change={(e) => handleDateChange(e, 'start')} />
  <input id="end-date" type="date" value={formatDateInput(endDate)} max="2025-07-01" on:change={(e) => handleDateChange(e, 'end')} />

        <RangeSlider 
          min={0} 
          max={maxAllowedIndex} 
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
          labelOverride={(domain, value) => labelForDomain(domain, value, isDefaultRemote)}
          descriptionOverride={(domain, value) => descriptionForDomain(domain, value, isDefaultRemote)}
        />
      {:else}
        {#if isLoading}
          <div class="progress-wrap">
            <div class="progress-header">
              {#if loadPhase === 'downloading'}
                Downloading CSV...
              {:else if loadPhase === 'parsing'}
                Parsing CSV...
              {:else}
                Loading...
              {/if}
            </div>
            {#if loadPhase === 'downloading'}
              {#if loadTotal}
                <div class="progress-bar"><div class="progress-fill" style="width: {loadProgress}%"></div></div>
                <div class="progress-label">{loadProgress}% ({formatBytes(loadBytes)} / {formatBytes(loadTotal)})</div>
              {:else}
                <div class="progress-bar indeterminate"></div>
                <div class="progress-label">{formatBytes(loadBytes)}</div>
              {/if}
            {:else}
              <div class="progress-bar indeterminate"></div>
            {/if}
          </div>
        {:else}
          <p>Waiting for data</p>
        {/if}
      {/if}
    </div>
  </div>
</div>

<style>
  /* Make the overall page non-scrollable */
  :global(html, body, #app) {
    height: 100%;
    overflow: hidden;
  }

  .container {
    padding: 1rem;
    display: flex;
    flex-direction: column;
    gap: 1rem;
    font-family: 'Inter', sans-serif;
    height: 100%; /* fill viewport */
    overflow: hidden; /* prevent page scroll */
  box-sizing: border-box; /* include padding in height to avoid clipping */
  }

  .title-section {
    text-align: center;
  }

  .title {
    font-size: 2rem;
    font-weight: 700;
    margin-bottom: 0.5rem;
  }

  /* removed unused .subtitle style */

  .content {
    display: flex;
    gap: 1.5rem;
    align-items: flex-start; /* Align items at the top */
  flex: 1;           /* take remaining height below the title */
  min-height: 0;     /* allow children to shrink */
  overflow: hidden;  /* no page scroll from content */
  }

  .filter-panel {
    background: #f9fafb;
    /* border-right: #969696 1px solid; */
    padding: 1.5rem;
    width: 300px; 
    display: flex;
    flex-direction: column;
    gap: 1rem;
    height: 100%;     /* fill the content column height */
    overflow: auto;   /* panel itself scrolls */
  box-sizing: border-box; /* keep padding within allotted height */
  -webkit-overflow-scrolling: touch; /* smoother scrolling on macOS/iOS */
  overscroll-behavior: contain; /* keep scroll events within the panel */
  }

  .filter-actions {
    display: flex;
    justify-content: flex-start;
  }
  .reset-btn {
    padding: 0.4rem 0.6rem;
    font-size: 0.85rem;
    background: #fff;
    border: 1px solid #ccc;
    border-radius: 4px;
    cursor: pointer;
  }
  .reset-btn:hover {
    background: #f7f7f7;
  }

  /* Only style top-level labels in the filter panel (avoid checkbox item labels) */
  .filter-panel > label {
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

  /* removed checkbox styles after switching to multi-select */

  .filter-panel input:focus,
  .filter-panel select:focus {
    outline: none;
    border-color: #4c8bf5;
  }

   .multi-select {
    min-height: 150px;
    overflow-y: auto;
    width: 100%; 
  }


  .scatterplot-container {
    flex: 1;          /* take remaining width next to panel */
    display: flex;
    justify-content: center;
    align-items: center;
    background: #fff;
    border-radius: 10px;
    height: 100%;     /* fill vertical space in content */
    min-height: 0;    /* allow to shrink with viewport */
    padding: 1rem;
    /* box-shadow: inset 0 0 10px rgba(0, 0, 0, 0.05); */
  box-sizing: border-box; /* ensure padding doesn't cause vertical overflow */
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

  /* Progress styles */
  .progress-wrap { display: flex; flex-direction: column; gap: 0.25rem; margin-bottom: 0.5rem; }
  .progress-header { font-size: 0.9rem; color: #333; font-weight: 600; }
  .progress-bar { position: relative; height: 8px; background: #e5e7eb; border-radius: 999px; overflow: hidden; }
  .progress-fill { height: 100%; background: #4c8bf5; width: 0%; transition: width 120ms linear; }
  .progress-bar.indeterminate::before {
    content: ""; position: absolute; left: -40%; width: 40%; height: 100%;
    background: #4c8bf5; animation: indet 1s infinite;
  }
  @keyframes indet { 0% { left: -40%; } 50% { left: 60%; } 100% { left: 100%; } }
  .progress-label { font-size: 0.8rem; color: #555; }
</style>