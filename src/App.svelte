<script>
  import { onMount } from "svelte";
  import Papa from "papaparse";
  import Scatterplot from "./components/Scatterplot.svelte";
  import RangeSlider from "./components/RangeSlider.svelte";
  import DetailCard from "./components/DetailCard.svelte";
  import { scaleOrdinal } from 'd3-scale';
  import { schemeCategory10 } from 'd3-scale-chromatic';

  // Resolve data URL from query params (url | filename [+ bucket]) or env fallback
  let resolvedDataUrl = "";
  $: isDefaultRemote = /^https?:\/\//i.test(resolvedDataUrl);
  function resolveDataUrl() {
    try {
      const params = new URLSearchParams(window.location.search);
      const directUrl = params.get("url");
      const filename = params.get("filename");
      const bucket = params.get("bucket");

      // Highest priority: full URL provided
      if (directUrl && /^https?:\/\//i.test(directUrl)) return directUrl;

      // Build from filename and (optional) bucket
      if (filename) {
        // Determine base from bucket param or env
        let base = "";
        if (bucket) {
          if (/^https?:\/\//i.test(bucket)) {
            base = bucket;
          } else if (bucket.includes(".")) {
            // Looks like a host (e.g. my-bucket.s3.amazonaws.com or custom domain)
            base = `https://${bucket}`;
          } else {
            // Treat as bare S3 bucket name
            base = `https://${bucket}.s3.amazonaws.com`;
          }
        } else {
          // Env-configured default base (e.g. https://my-bucket.s3.amazonaws.com/)
          base =
            import.meta.env.VITE_S3_BASE_URL ||
            import.meta.env.VITE_DATA_BASE_URL ||
            "https://pink-slime-public.s3.amazonaws.com/";
        }
        if (base && !base.endsWith("/")) base += "/";
        return base ? base + filename : filename;
      }

      // Fallbacks: explicit env or local file in /public
      return import.meta.env.VITE_DATA_URL || "data.csv";
    } catch (e) {
      console.warn("Failed to resolve data URL from query params:", e);
      return import.meta.env.VITE_DATA_URL || "data.csv";
    }
  }

  let data = [],
    columns = [],
    domainColumn = "org",
    uniqueValues = [];
  let selectedValues = new Set();
  let opacity = 0.02,
    startDate = null,
    endDate = null;
  let filteredData = [],
    allDates = [];
  let startDateIndex = 0;
  let endDateIndex = 0;
  let isPlaying = false;
  let playInterval = null;
  let highlightedData = [];
  $: highlightedSet = new Set(highlightedData.map((d) => d.id));

  let searchQuery = "";
  let showAnnotations = false;
  let hoveredData = null;
  let selectedData = null; // Pinned/clicked data

  // Display selected data if available, otherwise show hovered data
  $: displayedData = selectedData || hoveredData;

  // Color scale for the detail card
  $: colorScale = scaleOrdinal(schemeCategory10)
    .domain(domainColumn ? uniqueValues : []);

  // Computed min/max dates from actual data
  $: minDateFromData = allDates.length > 0 ? allDates[0] : null;
  $: maxDateFromData = allDates.length > 0 ? allDates[allDates.length - 1] : null;

  // Only allow org or cluster as color-by options
  $: allowedDomainColumns = columns.filter(
    (c) => c === "org" || c === "state",
  );
  $: {
    // Wait until columns are known before adjusting the selected domain
    if (!columns || columns.length === 0) {
      // do nothing until parsed
    } else if (
      allowedDomainColumns.length &&
      !allowedDomainColumns.includes(domainColumn)
    ) {
      domainColumn = allowedDomainColumns[0];
    } else if (allowedDomainColumns.length === 0) {
      domainColumn = "";
    }
  }

  // Loading/progress state
  let isLoading = false;
  let loadPhase = "idle"; // 'downloading' | 'parsing' | 'idle'
  let loadBytes = 0;
  let loadTotal = 0;
  let loadProgress = 0; // 0-100 when total known

  function formatBytes(bytes) {
    if (!bytes || bytes <= 0) return "0 KB";
    const units = ["B", "KB", "MB", "GB", "TB"];
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
      loadPhase = "downloading";
      loadBytes = 0;
      loadTotal = 0;
      loadProgress = 0;

      // Determine the final CSV URL once on mount
      resolvedDataUrl = resolveDataUrl();

      const response = await fetch(resolvedDataUrl, {
        mode: "cors",
        cache: "no-store",
      });
      if (!response.ok) throw new Error(`HTTP ${response.status}`);

      const len = response.headers.get("content-length");
      loadTotal = len ? parseInt(len, 10) : 0;

      let csvText = "";
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
        console.log("CSV head:", csvText.split("\n").slice(0, 5).join("\n"));
      } else {
        // Fallback without streaming/progress
        csvText = await response.text();
      }

      // Parse phase (indeterminate)
      loadPhase = "parsing";
      await Promise.resolve(parseCSV(csvText));
      loadPhase = "idle";
    } catch (err) {
      console.error("Failed to load CSV:", err);
    } finally {
      isLoading = false;
    }

    return () => {
      if (playInterval) clearInterval(playInterval);
    };
  });

  $: {
    // Determine if any filter is active
    const fullDateRange = allDates.length && startDateIndex === 0 && endDateIndex === allDates.length - 1;
    const hasSelection = selectedValues.size > 0 && selectedValues.size < uniqueValues.length;
    const hasSearch = searchQuery && searchQuery.trim().length > 0;
    const anyFilterActive = !fullDateRange || hasSelection || hasSearch;

    filteredData = data.map((d) => {
      const inDateRange = (!startDate || d.date >= startDate) && (!endDate || d.date <= endDate);
      const inSelection = hasSelection ? selectedValues.has(d[domainColumn]) : true;
      let inSearch = true;
      if (hasSearch) {
        try {
          const regex = new RegExp(searchQuery, "i");
          inSearch = regex.test(d.title ?? "") || regex.test(d.text ?? "");
        } catch {
          inSearch = (d.title ?? "").toLowerCase().includes(searchQuery.toLowerCase()) ||
                     (d.text ?? "").toLowerCase().includes(searchQuery.toLowerCase());
        }
      }
      let isActive = false;
      let isHighlighted = false;
      if (anyFilterActive) {
        isActive = inDateRange && inSelection && inSearch;
        isHighlighted = inDateRange && inSelection && inSearch;
      } else {
        isActive = false;
        isHighlighted = false;
      }
      return {
        ...d,
        isActive,
        isHighlighted,
      };
    });
  }

  $: startPercent =
    allDates.length > 1 ? (startDateIndex / (allDates.length - 1)) * 100 : 0;
  $: endPercent =
    allDates.length > 1 ? (endDateIndex / (allDates.length - 1)) * 100 : 100;

  // Use actual max date from data instead of arbitrary cap
  $: maxAllowedIndex = allDates.length > 0 ? allDates.length - 1 : 0;

  function parseCSV(csvText) {
    // Lowercase headers before parsing
    const lines = csvText.split(/\r?\n/);
    if (lines.length > 0) {
      const headerLine = lines[0];
      const lowerHeader = headerLine
        .split(",")
        .map((h) => h.trim().toLowerCase())
        .join(",");
      lines[0] = lowerHeader;
      csvText = lines.join("\n");
    }
    const result = Papa.parse(csvText, { header: true });
    data = result.data
      .filter((d) => d.x && d.y && d.date)
      .map((d, i) => ({
        ...d,
        x: +d.x,
        y: +d.y,
        date: new Date(d.date),
        id: i,
      }));

    columns = result.meta.fields || [];
    allDates = [...new Set(data.map((d) => d.date.getTime()))]
      .sort((a, b) => a - b)
      .map((t) => new Date(t));

    startDate = allDates[0];
    endDate = allDates[allDates.length - 1];
    startDateIndex = 0;
    endDateIndex = allDates.length - 1;
    updateDateIndices();

    if (domainColumn) {
      uniqueValues = [
        ...new Set(
          data
            .map((d) => d[domainColumn])
            .filter((v) => v !== undefined && v !== null && v !== ""),
        ),
      ].sort((a, b) => String(a).localeCompare(String(b)));
    }
  }

  function handleDomainChange(event) {
    domainColumn = event.target.value;
    uniqueValues = domainColumn
      ? [...new Set(data.map((d) => d[domainColumn]).filter(Boolean))].sort(
          (a, b) => String(a).localeCompare(String(b)),
        )
      : [];
    selectedValues = new Set();
    showAnnotations = false;
  }

  function handleSelectionChange(event) {
    const selectedOptions = [...event.target.selectedOptions].map(
      (o) => o.value,
    );

    // Check if all values are selected
    const allSelected = selectedOptions.length === uniqueValues.length;

    if (allSelected || selectedOptions.length === 0) {
      // If all values are selected or none are selected, clear the selection
      selectedValues = new Set();
      highlightedData = [];
    } else {
      // Only update selection if a subset is chosen
      selectedValues = new Set(selectedOptions);
      highlightedData = data.filter(
        (d) =>
          selectedValues.has(d[domainColumn]) &&
          (!startDate || d.date >= startDate) &&
          (!endDate || d.date <= endDate),
      );
    }

    showAnnotations = false;
  }

  // Keep highlightedData in sync is handled by selection handlers for multi-select

  function updateSelectedDates(start, end, fromIndices = false) {
    if (fromIndices) {
      startDate = allDates[start] || allDates[0];
      endDate = allDates[end] || allDates[allDates.length - 1];
    } else {
      startDate = start;
      endDate = end;
    }
    showAnnotations = false;
    updateDateIndices();
  }

  function updateDateIndices() {
    if (startDate) {
      const timestamp = startDate.getTime();
      startDateIndex = Math.max(
        0,
        allDates.findIndex((d) => d.getTime() >= timestamp),
      );
    } else {
      startDateIndex = 0;
    }

    if (endDate) {
      const timestamp = endDate.getTime();
      let ei = -1;
      for (let i = allDates.length - 1; i >= 0; i--) {
        if (allDates[i].getTime() <= timestamp) {
          ei = i;
          break;
        }
      }
      endDateIndex = ei >= 0 ? ei : allDates.length - 1;
    } else {
      endDateIndex = allDates.length - 1;
    }

    if (startDateIndex > endDateIndex) startDateIndex = endDateIndex;
    if (endDateIndex < startDateIndex) endDateIndex = startDateIndex;
  }

  function handleDateChange(e, type) {
    if (e.detail !== undefined) {
      // Handle slider events
      if (type === "start") {
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

      if (type === "start") {
        updateSelectedDates(newDate, endDate);
      } else {
        updateSelectedDates(startDate, newDate);
      }

      // Update indices based on the new dates
      updateDateIndices();
    }
  }

  function formatDateInput(date) {
    return date ? date.toISOString().split("T")[0] : "";
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

    // Handle boundary conditions
    if (newStartIndex < 0) {
      newStartIndex = 0;
      newEndIndex = Math.min(rangeDuration, allDates.length - 1);
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
    const defaultDomain = columns.includes("org")
      ? "org"
      : allowedDomainColumns[0] || "";
    domainColumn = defaultDomain;

    // Clear selection and highlights
    selectedValues = new Set();
    highlightedData = [];

    // Reset search and annotations
    searchQuery = "";
    showAnnotations = false;

    // Reset opacity to initial default
    opacity = 0.02;

    // Force Svelte to update opacity binding
    opacity = +opacity;

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
      Each dot represents an article. When the dots are closer together, the
      articles are similar in meaning.
    </p>
  </div>

  <div class="content">
    <!-- Filters Panel -->
    <div class="filter-panel left-panel">
      <div class="filter-actions">
        <button class="reset-btn" on:click={resetFilters}>Reset filters</button>
      </div>

      <div class="nerd-box">
        <details>
          <summary><span class="toggle-icon">+</span> What is a Semantic Map?</summary>
          <div class="nerd-box-content">
            <p>
              Semantic maps are tools that allow users to visually explore how
              different topics and ideas are connected. By analyzing the
              relationships between articles, these maps can reveal patterns and
              clusters in the data.
            </p>
            <p>Newsrooms can use semantic maps to:</p>
            <ul>
              <li>
                Audit their coverage by identifying themes that may be
                underrepresented or overlooked.
              </li>
              <li>Track how their editorial focus evolves over time.</li>
              <li>
                Discover which topics are being covered by other outlets,
                offering opportunities to adjust their editorial focus or
                collaborate.
              </li>
            </ul>
          </div>
        </details>
      </div>

      <label for="file-upload">📁 Upload CSV:</label>
      <input
        id="file-upload"
        type="file"
        accept=".csv"
        on:change={handleFileUpload}
      />

      <div class="info-box">
        <details>
          <summary><span class="toggle-icon">+</span> CSV File Requirements</summary>
          <div class="info-box-content">
            <p>Your CSV file must include the following columns:</p>
            <ul>
              <li><strong>x, y</strong> - Numeric coordinates for positioning points on the scatterplot</li>
              <li><strong>date</strong> - Date field for timeline filtering (ISO format recommended)</li>
              <li><strong>title</strong> - Article headline or title (searchable)</li>
              <li><strong>text</strong> - Article body or description (searchable)</li>
              <li><strong>org or state</strong> - Categorical field for color-coding points (at least one required)</li>
            </ul>
            <p><strong>Optional columns:</strong></p>
            <ul>
              <li><strong>url, link, href, or permalink</strong> - Clickable link (Ctrl/Cmd + Click to open)</li>
            </ul>
            <p><em>Note: Column names are case-insensitive.</em></p>
          </div>
        </details>
      </div>

      <label for="search-input">🔍 Search Text (supports regex):</label>
      <input
        id="search-input"
        type="text"
        placeholder="Search or regex pattern..."
        on:input={handleSearch}
      />

      {#if allowedDomainColumns.length}
        <label for="domain-column">🎨 Color by Column:</label>
        <select
          id="domain-column"
          on:change={handleDomainChange}
          bind:value={domainColumn}
        >
          <option value="" disabled>Select column</option>
          {#each allowedDomainColumns as column}
            <option value={column}>{column}</option>
          {/each}
        </select>
      {/if}

      {#if uniqueValues.length}
        <label for="value-select"
          >✨ Highlight Values (hold Shift/Cmd to select multiple):</label
        >
        <select
          id="value-select"
          multiple
          size="5"
          class="multi-select"
          on:change={handleSelectionChange}
        >
          {#each uniqueValues as item}
            <option value={item} selected={selectedValues.has(item)}
              >{item}</option
            >
          {/each}
        </select>
      {/if}

      <label for="opacity-slider">💡 Adjust Opacity:</label>
      <input
        id="opacity-slider"
        type="range"
        min="0"
        max=".3"
        step="0.01"
        bind:value={opacity}
        on:input={handleOpacityChange}
      />

      <div class="date-controls">
        <label for="start-date">📅 Date Range:</label>
        <input
          id="start-date"
          type="date"
          value={formatDateInput(startDate)}
          max={formatDateInput(maxDateFromData)}
          on:change={(e) => handleDateChange(e, "start")}
        />
        <input
          id="end-date"
          type="date"
          value={formatDateInput(endDate)}
          max={formatDateInput(maxDateFromData)}
          on:change={(e) => handleDateChange(e, "end")}
        />

        <RangeSlider
          min={0}
          max={maxAllowedIndex}
          bind:startValue={startDateIndex}
          bind:endValue={endDateIndex}
          on:startChange={(e) => handleDateChange(e, "start")}
          on:endChange={(e) => handleDateChange(e, "end")}
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
          uniqueValues={uniqueValues}
          bind:hoveredData
          bind:selectedData
        />
      {:else if isLoading}
        <div class="progress-wrap">
          <div class="progress-header">
            {#if loadPhase === "downloading"}
              Downloading CSV...
            {:else if loadPhase === "parsing"}
              Parsing CSV...
            {:else}
              Loading...
            {/if}
          </div>
          {#if loadPhase === "downloading"}
            {#if loadTotal}
              <div class="progress-bar">
                <div class="progress-fill" style="width: {loadProgress}%"></div>
              </div>
              <div class="progress-label">
                {loadProgress}% ({formatBytes(loadBytes)} / {formatBytes(
                  loadTotal,
                )})
              </div>
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
    </div>

    <!-- Right Panel for Speech Details -->
    <div class="detail-panel">
      <DetailCard
        hoveredData={displayedData}
        {data}
        {domainColumn}
        {colorScale}
        {searchQuery}
        isPinned={!!selectedData}
        on:unpin={() => selectedData = null}
      />
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
    font-family: "Inter", sans-serif;
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
    flex: 1; /* take remaining height below the title */
    min-height: 0; /* allow children to shrink */
    overflow: hidden; /* no page scroll from content */
  }

  .filter-panel {
    background: #f9fafb;
    padding: 1.5rem;
    width: 300px;
    display: flex;
    flex-direction: column;
    gap: 1rem;
    height: 100%; /* fill the content column height */
    overflow: auto; /* panel itself scrolls */
    box-sizing: border-box; /* keep padding within allotted height */
    -webkit-overflow-scrolling: touch; /* smoother scrolling on macOS/iOS */
    overscroll-behavior: contain; /* keep scroll events within the panel */
  }

  .detail-panel {
    background: #f9fafb;
    padding: 1.5rem;
    width: 450px;
    display: flex;
    flex-direction: column;
    height: 100%;
    overflow: auto;
    box-sizing: border-box;
    -webkit-overflow-scrolling: touch;
    overscroll-behavior: contain;
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
    flex: 1; /* take remaining width next to panel */
    display: flex;
    justify-content: center;
    align-items: center;
    background: #fff;
    border-radius: 10px;
    height: 100%; /* fill vertical space in content */
    min-height: 0; /* allow to shrink with viewport */
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
    font-weight: bold;
    cursor: pointer;
    list-style: none;
    display: flex;
    align-items: center;
    gap: 0.5rem;
    user-select: none;
  }

  .nerd-box summary:hover {
    opacity: 0.8;
  }

  .nerd-box details[open] summary {
    margin-bottom: 0.75rem;
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

  .info-box {
    background: #eef5ff;
    border: 1px solid #4c8bf5;
    padding: 1rem;
    border-radius: 8px;
    font-size: 0.9rem;
    line-height: 1.6;
    margin-bottom: 1rem;
  }

  .info-box summary {
    font-weight: bold;
    cursor: pointer;
    list-style: none;
    display: flex;
    align-items: center;
    gap: 0.5rem;
    user-select: none;
  }

  .info-box summary:hover {
    opacity: 0.8;
  }

  .info-box details[open] summary {
    margin-bottom: 0.75rem;
  }

  .info-box-content p {
    margin-bottom: 0.75rem;
    color: #444;
  }

  .info-box-content ul {
    list-style-type: disc;
    padding-left: 1.5rem;
    margin-bottom: 0.75rem;
  }

  .info-box-content li {
    margin-bottom: 0.4rem;
    color: #444;
  }

  .info-box-content strong {
    color: #1565c0;
  }

  .info-box .toggle-icon {
    background: rgba(76, 139, 245, 0.25);
    color: #1565c0;
  }

  /* Toggle icon styling */
  .toggle-icon {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 22px;
    height: 22px;
    border-radius: 4px;
    background: rgba(0, 0, 0, 0.1);
    font-weight: bold;
    font-size: 1.1rem;
    line-height: 1;
    flex-shrink: 0;
    transition: all 0.2s ease;
    position: relative;
  }

  .nerd-box .toggle-icon {
    background: rgba(76, 139, 245, 0.25);
    color: #1565c0;
  }

  /* Hide the default + text and use ::after for dynamic content */
  .toggle-icon {
    font-size: 0;
  }

  .toggle-icon::after {
    content: '+';
    font-size: 1.1rem;
    font-weight: bold;
  }

  details[open] .toggle-icon::after {
    content: '−';
  }

  summary:hover .toggle-icon {
    transform: scale(1.1);
  }

  /* Progress styles */
  .progress-wrap {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
    margin-bottom: 0.5rem;
  }
  .progress-header {
    font-size: 0.9rem;
    color: #333;
    font-weight: 600;
  }
  .progress-bar {
    position: relative;
    height: 8px;
    background: #e5e7eb;
    border-radius: 999px;
    overflow: hidden;
  }
  .progress-fill {
    height: 100%;
    background: #4c8bf5;
    width: 0%;
    transition: width 120ms linear;
  }
  .progress-bar.indeterminate::before {
    content: "";
    position: absolute;
    left: -40%;
    width: 40%;
    height: 100%;
    background: #4c8bf5;
    animation: indet 1s infinite;
  }

  #opacity-slider {
    padding: 0em !important;
  }

  @keyframes indet {
    0% {
      left: -40%;
    }
    50% {
      left: 60%;
    }
    100% {
      left: 100%;
    }
  }
  .progress-label {
    font-size: 0.8rem;
    color: #555;
  }
</style>
