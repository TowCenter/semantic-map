<script>
  import { onMount } from 'svelte';
  import { scaleOrdinal } from 'd3-scale';
  import { schemeCategory10 } from 'd3-scale-chromatic';

  export let hoveredData;
  export let domainColumn;
  export let data;
  export let colorScale;
  export let posX = 0; // absolute position inside chart container
  export let posY = 0;
  
  let filteredData = [];
  let dates = [];
  let selectedDate = null;
  let isPlaying = false;
  let interval


  // const colorScale = scaleOrdinal(schemeCategory10);

  onMount(() => {
      colorScale.domain(data.map(d => d[domainColumn]));
  });
  
</script>

<div class="detail-card" style="left: {posX}px; top: {posY}px;">
  {#if hoveredData}
  <h1>{hoveredData.title}</h1>
  <span style="background: {colorScale(hoveredData[domainColumn])};">
      {hoveredData[domainColumn]}</span>
  <h2>{hoveredData.date.toISOString().split('T')[0]}</h2>  
  <p> {hoveredData.text}</p>
  {:else}
  <p>Hover over a circle to see details here.</p>
  {/if}
</div>

<style>
  .detail-card {
    position: absolute;
    padding: 10px 12px;
    border: 1px solid rgba(0, 0, 0, 0.15);
    border-radius: 8px;
    background: rgba(255, 255, 255, 0.98);
    box-shadow: 0 8px 24px rgba(0,0,0,0.15);
    max-width: 280px;
    max-height: 50vh;
    overflow-y: auto;
    line-height: 1.45;
    pointer-events: none; /* don't block hover on canvas */
    z-index: 10;
  }


  h1,h2 {
      margin: 0;
      padding: 0;
      font-weight: 300;
      margin-bottom: 10px;
      }

  h1 {
      font-size: 0.95rem;
      font-weight: 600;
      margin-bottom: 6px;
      }

  h2 {
      font-size: 0.78rem;
      text-transform: uppercase;
      color: #555;
      }

  span {
      padding: 4px 6px;
      display: inline-block;
      vertical-align: bottom;
      border-radius: 4px;
      color: white;
      margin-bottom: 8px;
      font-size: 0.75rem;
      }

  p {
      font-size: 0.85rem;
      font-weight: 400;
      margin: 0;
    }
</style>