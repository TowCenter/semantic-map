<script>
  import { onMount, createEventDispatcher } from 'svelte';
  import { scaleOrdinal } from 'd3-scale';
  import { schemeCategory10 } from 'd3-scale-chromatic';

  const dispatch = createEventDispatcher();

  export let hoveredData;
  export let domainColumn;
  export let data;
  export let colorScale;
  export let searchQuery = "";
  export let isPinned = false;
  export let labelOverride = null; // optional function(domain, value) -> label
  export let descriptionOverride = null; // optional function(domain, value) -> description

  let filteredData = [];
  let dates = [];
  let selectedDate = null;
  let isPlaying = false;
  let interval;

  onMount(() => {
      colorScale.domain(data.map(d => d[domainColumn]));
  });

  // Function to highlight search terms in text
  function highlightText(text, query) {
    if (!query || !text) return text;

    try {
      // Try to use the query as a regex pattern
      const regex = new RegExp(`(${query})`, 'gi');
      return text.replace(regex, '<mark>$1</mark>');
    } catch (e) {
      // If regex is invalid, fall back to simple string search
      const escapedQuery = query.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
      const regex = new RegExp(`(${escapedQuery})`, 'gi');
      return text.replace(regex, '<mark>$1</mark>');
    }
  }

</script>

<div class="detail-card">
  {#if hoveredData}
    {#if isPinned}
      <div class="pin-header">
        <span class="pin-indicator">📌 Pinned</span>
        <button class="unpin-btn" on:click={() => dispatch('unpin')}>✕</button>
      </div>
    {/if}
    <h1>{@html highlightText(hoveredData.title, searchQuery)}</h1>
    <span style="background: {colorScale(hoveredData[domainColumn])};">
      {labelOverride ? labelOverride(domainColumn, hoveredData[domainColumn]) : hoveredData[domainColumn]}
    </span>
    <h2>{hoveredData.date.toISOString().split('T')[0]}</h2>
    {#if descriptionOverride}
      {#if descriptionOverride(domainColumn, hoveredData[domainColumn])}
        <p><em>{descriptionOverride(domainColumn, hoveredData[domainColumn])}</em></p>
      {/if}
    {/if}
    <p>{@html highlightText(hoveredData.text, searchQuery)}</p>
  {:else}
    <p class="placeholder-text">{isPinned ? 'Click on a circle to pin it here.' : 'Hover over a circle to see details here.'}</p>
  {/if}
</div>

<style>
  .detail-card {
    padding: 0;
    border-radius: 8px;
    background: #fff;
    height: 100%;
    overflow-y: auto;
    line-height: 1.5;
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
  }

  h1,h2 {
    margin: 0;
    padding: 0;
    font-weight: 300;
  }

  h1 {
    font-size: 1.1rem;
    font-weight: 600;
    line-height: 1.4;
  }

  h2 {
    font-size: 0.85rem;
    text-transform: uppercase;
    color: #555;
  }

  span {
    padding: 4px 8px;
    display: inline-block;
    vertical-align: bottom;
    border-radius: 4px;
    color: white;
    font-size: 0.8rem;
    width: fit-content;
  }

  p {
    font-size: 0.9rem;
    font-weight: 400;
    margin: 0;
    line-height: 1.6;
  }

  .placeholder-text {
    color: #999;
    font-style: italic;
  }

  /* Yellow highlighting for search terms */
  :global(.detail-card mark) {
    background-color: yellow;
    color: black;
    padding: 0 2px;
    border-radius: 2px;
  }

  .pin-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 0.5rem;
    background: #e3f2fd;
    border-radius: 6px;
    margin-bottom: 0.5rem;
  }

  .pin-indicator {
    font-size: 0.85rem;
    font-weight: 600;
    color: #1976d2;
  }

  .unpin-btn {
    background: transparent;
    border: none;
    font-size: 1.2rem;
    cursor: pointer;
    color: #666;
    padding: 0;
    width: 24px;
    height: 24px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 4px;
  }

  .unpin-btn:hover {
    background: rgba(0, 0, 0, 0.1);
    color: #333;
  }
</style>