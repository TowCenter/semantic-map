<script>
    import { onMount } from 'svelte';
    import { scaleLinear, scaleOrdinal } from 'd3-scale';
    import { max, min } from 'd3-array';
  import DetailCard from './DetailCard.svelte';
  import { schemeCategory10 } from 'd3-scale-chromatic';
  import { select, zoom, zoomIdentity } from 'd3';
    
    export let data = [];
    export let domainColumn = "";
    export let opacity = 1;
    export let selectedValues = new Set();
    export let searchQuery = ""; 
    export let showAnnotations = true;
    export let highlightedData = [];
    export let startDate = null;  // Add these new props
    export let endDate = null;    // Add these new props


    let annotations = [
        { x: 500, y: 400, radius: 30, label: "Border Belt Independent and North Carolina Coastal Federation have both published about GenX — an industrial chemical.", label_x: -400, label_y: -80 },
        { x: 720, y: 170, radius: 50, label: "Only Carolina Peacemaker seems to be publishing about personal health.", label_x: -100, label_y: 130 }
    ];

  let canvas;
  let containerEl; // wrapper element whose size controls the canvas
    let ctx;
    
    let containerWidth = 800;
    let containerHeight = 600;
    
    const margin = { top: 20, right: 20, bottom: 20, left: 20 };
    const radius = 6;
    
  let hoveredData = null;
  let lastHoveredData = null;
  let t = zoomIdentity; // d3-zoom transform
  const MIN_SCALE = 0.5;
  const MAX_SCALE = 10;
    $: highlightedSet = new Set(highlightedData.map(d => d.id));
  $: uniqueDomainCount = new Set(data.map(d => d[domainColumn])).size;

    // Tooltip position in CSS pixels within the chart container
    let tooltipX = 0;
    let tooltipY = 0;
    const TOOLTIP_MAX_W = 280; // px, used for simple boundary checks
    const TOOLTIP_H = 160;     // approximate height for clamping


    
    $: innerWidth = containerWidth - margin.left - margin.right;
    $: innerHeight = containerHeight - margin.top - margin.bottom;
    
    // Guard scales against degenerate domains
    $: xDomainRaw = [min(data, d => d.x), max(data, d => d.x)];
    $: xDomain = (xDomainRaw[0] === xDomainRaw[1])
      ? [xDomainRaw[0] - 1, xDomainRaw[1] + 1]
      : xDomainRaw;
    $: xScale = scaleLinear()
      .domain(xDomain)
      .range([0, innerWidth]);
    
    $: yDomainRaw = [min(data, d => d.y), max(data, d => d.y)];
    $: yDomain = (yDomainRaw[0] === yDomainRaw[1])
      ? [yDomainRaw[0] - 1, yDomainRaw[1] + 1]
      : yDomainRaw;
    $: yScale = scaleLinear()
      .domain(yDomain)
      .range([innerHeight, 0]);
    
    // Reactive color scale to follow data and domainColumn changes
    let colorScale;
    $: colorScale = scaleOrdinal(schemeCategory10)
      .domain(
        domainColumn
          ? [...new Set(data.map(d => d[domainColumn]).filter(v => v !== undefined && v !== null && v !== ''))]
          : []
      );

    // Precompute date bounds and whether full range is selected
    $: minDate = data.length ? new Date(Math.min(...data.map(d => d.date.getTime()))) : null;
    $: maxDate = data.length ? new Date(Math.max(...data.map(d => d.date.getTime()))) : null;
    $: isFullDateRange = !!(
      startDate && endDate && minDate && maxDate &&
      startDate.getTime() === minDate.getTime() &&
      endDate.getTime() === maxDate.getTime()
    );
    
    // HiDPI setup and responsive sizing based on container element
    let dpr = 1;
    function setupCanvasDPI() {
      if (!canvas || !containerEl) return;
      const rect = containerEl.getBoundingClientRect();
      // Keep containerWidth/Height in CSS pixels to sync scales and mouse events
      if (rect.width > 0 && rect.height > 0) {
        containerWidth = Math.floor(rect.width);
        containerHeight = Math.floor(rect.height);
      }
      dpr = Math.max(window.devicePixelRatio || 1, 1);
      // Visually fill container; internal pixel size scaled for crispness
      canvas.style.width = '100%';
      canvas.style.height = '100%';
      canvas.width = Math.max(1, Math.floor(containerWidth * dpr));
      canvas.height = Math.max(1, Math.floor(containerHeight * dpr));
      ctx = canvas.getContext('2d');
      ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
    }

    function draw() {
      if (!ctx || !data.length) return;

      ctx.clearRect(0, 0, containerWidth, containerHeight);
      ctx.save();
      // Apply d3 zoom/pan transform
      ctx.translate(t.x, t.y);
      ctx.scale(t.k, t.k);

      // Draw data points
      // data.forEach(d => {
      //   const matchesSearch = searchQuery && d.title?.toLowerCase().includes(searchQuery.toLowerCase());
      //   const isHighlighted = highlightedSet.has(d.id) || matchesSearch;
      //   const isSelected = selectedValues.has(d[domainColumn]);
      //   const isInDateRange = (!startDate || d.date >= startDate) && 
      //                        (!endDate || d.date <= endDate);

      //   ctx.beginPath();
      //   ctx.arc(margin.left + xScale(d.x), margin.top + yScale(d.y), radius, 0, Math.PI * 2);
      //   ctx.fillStyle = colorScale(d[domainColumn]);
        
      //   // Set opacity based on conditions
      //   if (isInDateRange && !isFullDateRange) {
      //     ctx.globalAlpha = 1;
      //   } else if ((isHighlighted || isSelected) && isFullDateRange) {
      //     ctx.globalAlpha = 1;
      //   } else {
      //     ctx.globalAlpha = opacity * 0.2;
      //   }

      //   ctx.fill();

      // });

      data.forEach(d => {
        const matchesSearch = searchQuery && (
          d.text?.toLowerCase().includes(searchQuery.toLowerCase()) ||
          d.title?.toLowerCase().includes(searchQuery.toLowerCase())
        );
        const isHighlighted = highlightedSet.has(d.id) || matchesSearch;
        const isSelected = selectedValues.has(d[domainColumn]);
        const isInDateRange = (!startDate || d.date >= startDate) && 
                            (!endDate || d.date <= endDate);

        ctx.beginPath();
        // Keep point radius constant in screen pixels by compensating for zoom
        ctx.arc(margin.left + xScale(d.x), margin.top + yScale(d.y), Math.max(0.5, radius / t.k), 0, Math.PI * 2);
        ctx.fillStyle = colorScale(d[domainColumn]);
        
        // Set opacity based on conditions
        if ((isHighlighted || isSelected) && isInDateRange && !isFullDateRange) {
          // Show points that match both filters at full opacity
          ctx.globalAlpha = 1;
        } else if (isInDateRange && !isFullDateRange && selectedValues.size === 0) {
          // If only date range is active (no highlights selected), show all points in range
          ctx.globalAlpha = 1;
        } else if ((isHighlighted || isSelected) && isFullDateRange) {
          // If only highlights are active (full date range), show highlighted points
          ctx.globalAlpha = 1;
        } else {
          // Dim all other points
          ctx.globalAlpha = opacity * 0.2;
        }
        
  ctx.fill();
  ctx.globalAlpha = 1; // reset for next operations
      });


      // Draw annotations only if showAnnotations is true
      if (showAnnotations) {
        annotations.forEach(annotation => {
          ctx.globalAlpha = 1;

          // Draw annotation circle
          ctx.beginPath();
          ctx.arc(annotation.x, annotation.y, annotation.radius, 0, Math.PI * 2);
          ctx.strokeStyle = "red";
          ctx.lineWidth = 2;
          ctx.setLineDash([5, 5]);
          ctx.stroke();

          // Set text properties for measuring
          const maxWidth = 180; // Maximum width for the label
          const lineHeight = 12; // Line height for wrapped text
          ctx.font = "16px Arial";
          
          // Calculate label bounding box
          const labelX = annotation.x + annotation.label_x;
          const labelY = annotation.y + annotation.label_y;
          
          // Determine text bounds by measuring and wrapping text
          let textWidth = 0;
          let textHeight = 0;
          let lines = [];
          let currentLine = "";
          
          // Process text wrapping to calculate height and width
          if (annotation.label) {
            const words = annotation.label.split(" ");
            let line = "";
            
            words.forEach((word, index) => {
              const testLine = line + word + " ";
              const testWidth = ctx.measureText(testLine).width;
              
              if (testWidth > maxWidth && index > 0) {
                lines.push(line);
                textWidth = Math.max(textWidth, ctx.measureText(line).width);
                line = word + " ";
              } else {
                line = testLine;
              }
            });
            
            lines.push(line);
            textWidth = Math.max(textWidth, ctx.measureText(line).width);
            textHeight = lineHeight * lines.length;
          }
          
          // Label rectangle bounds
          const rectX = labelX;
          const rectY = labelY - lineHeight; // Offset to account for text baseline
          const rectWidth = textWidth;
          const rectHeight = textHeight;
          
          // Find closest point on the rectangle to the circle center
          // First determine which side of the rectangle is closest to the circle center
          let closestX, closestY;
          
          // Calculate x-coordinate of closest point
          if (annotation.x < rectX) {
            closestX = rectX;
          } else if (annotation.x > rectX + rectWidth) {
            closestX = rectX + rectWidth;
          } else {
            closestX = annotation.x;
          }
          
          // Calculate y-coordinate of closest point
          if (annotation.y < rectY) {
            closestY = rectY;
          } else if (annotation.y > rectY + rectHeight) {
            closestY = rectY + rectHeight;
          } else {
            closestY = annotation.y;
          }
          
          // Calculate the starting point of the line on the circle's outline
          const angle = Math.atan2(closestY - annotation.y, closestX - annotation.x);
          const startX = annotation.x + annotation.radius * Math.cos(angle);
          const startY = annotation.y + annotation.radius * Math.sin(angle);

          // Draw line connecting the circle outline to the closest point on the rectangle
          ctx.setLineDash([]); // Solid line for the connector
          ctx.beginPath();
          ctx.moveTo(startX, startY);
          ctx.lineTo(closestX, closestY);
          ctx.strokeStyle = "red";
          ctx.lineWidth = 1;
          ctx.stroke();

          // Draw label with wrapping
          if (annotation.label) {
            ctx.font = "16px Arial";
            ctx.fillStyle = "red";
            // increase line height between lines of t3ext
            const lineHeight = 16; // Adjusted line height for better readability

            // Draw each line of text
            lines.forEach((line, index) => {
              ctx.fillText(line, labelX, labelY + index * lineHeight);
            });
          }
        });
      }

      ctx.setLineDash([]); // Reset line dash

  if (hoveredData) {
        const baseX = margin.left + xScale(hoveredData.x);
        const baseY = margin.top + yScale(hoveredData.y);

        // Draw the highlighted point on top
        ctx.beginPath();
        ctx.arc(baseX, baseY, Math.max(0.5, radius / t.k), 0, Math.PI * 2);
        ctx.fillStyle = colorScale(hoveredData[domainColumn]);
        ctx.globalAlpha = 1;
        ctx.fill();
        ctx.strokeStyle = 'black';
        ctx.lineWidth = Math.max(1, 2 / t.k);
        ctx.stroke();

        // Compute tooltip screen position taking zoom into account
        const [zx, zy] = [t.applyX(baseX), t.applyY(baseY)];

        // Prefer to the right; flip if near right edge
  let tx = zx + 12; // offset from point
        if (tx + TOOLTIP_MAX_W > containerWidth - margin.right) {
          tx = zx - 12 - TOOLTIP_MAX_W;
        }

        // Vertical placement and clamping
        let ty = zy - TOOLTIP_H * 0.5; // center vertically around point
        const topBound = margin.top + 8;
        const bottomBound = containerHeight - margin.bottom - (TOOLTIP_H + 8);
        ty = Math.max(topBound, Math.min(ty, bottomBound));

        tooltipX = tx;
        tooltipY = ty;
      }

      ctx.restore();
    }
    
  function handleMouseMove(event) {
      const rect = canvas.getBoundingClientRect();
      
      // Calculate scaling ratio between internal canvas dimensions and displayed dimensions
      const scaleX = containerWidth / rect.width;
      const scaleY = containerHeight / rect.height;
      
  // Adjust mouse coordinates based on the scaling ratio (canvas pixel coords)
  const mouseX = (event.clientX - rect.left) * scaleX;
  const mouseY = (event.clientY - rect.top) * scaleY;
    
  // Inverse transform via d3-zoom
  const [adjustedX, adjustedY] = t.invert([mouseX, mouseY]);
    
      const foundData = data.find(d => {
        const worldX = margin.left + xScale(d.x);
        const worldY = margin.top + yScale(d.y);
        const dx = worldX - adjustedX;
        const dy = worldY - adjustedY;
        const isInRange = Math.sqrt(dx * dx + dy * dy) < (radius + 3) / t.k;

        // Check if point is currently highlighted based on filters
        const matchesSearch = searchQuery && (
          d.text?.toLowerCase().includes(searchQuery.toLowerCase()) ||
          d.title?.toLowerCase().includes(searchQuery.toLowerCase())
        );
        const isHighlighted = highlightedSet.has(d.id) || matchesSearch;
        const isSelected = selectedValues.has(d[domainColumn]);
        const isInDateRange = (!startDate || d.date >= startDate) && 
                             (!endDate || d.date <= endDate);

        // Allow hovering in default state or if point matches filters
        const isDefaultState = selectedValues.size === 0 || 
                             selectedValues.size === uniqueDomainCount;
        const isVisible = isDefaultState || 
                         ((isHighlighted || isSelected) && isInDateRange && !isFullDateRange) ||
                         (isInDateRange && !isFullDateRange && selectedValues.size === 0) ||
                         ((isHighlighted || isSelected) && isFullDateRange);

        return isInRange && isVisible;
      });

  if (foundData) {
        hoveredData = foundData;
        lastHoveredData = foundData;
        // indicate interactivity
        canvas.style.cursor = 'pointer';
      } else {
        hoveredData = null;
        lastHoveredData = null;
        canvas.style.cursor = 'crosshair';
      }

      draw();
    }

    function handleClick(event) {
      const rect = canvas.getBoundingClientRect();
      const scaleX = containerWidth / rect.width;
      const scaleY = containerHeight / rect.height;
  const mouseX = (event.clientX - rect.left) * scaleX;
  const mouseY = (event.clientY - rect.top) * scaleY;
      const [adjustedX, adjustedY] = t.invert([mouseX, mouseY]);

      const foundData = data.find(d => {
        const worldX = margin.left + xScale(d.x);
        const worldY = margin.top + yScale(d.y);
        const dx = worldX - adjustedX;
        const dy = worldY - adjustedY;
        // Keep hit radius in screen pixels by scaling threshold by 1/k
        const isInRange = Math.sqrt(dx * dx + dy * dy) < (radius + 3) / t.k;

        const matchesSearch = searchQuery && (
          d.text?.toLowerCase().includes(searchQuery.toLowerCase()) ||
          d.title?.toLowerCase().includes(searchQuery.toLowerCase())
        );
        const isHighlighted = highlightedSet.has(d.id) || matchesSearch;
        const isSelected = selectedValues.has(d[domainColumn]);
        const isInDateRange = (!startDate || d.date >= startDate) && 
                             (!endDate || d.date <= endDate);

        const isDefaultState = selectedValues.size === 0 || 
                             selectedValues.size === uniqueDomainCount;
        const isVisible = isDefaultState || 
                         ((isHighlighted || isSelected) && isInDateRange && !isFullDateRange) ||
                         (isInDateRange && !isFullDateRange && selectedValues.size === 0) ||
                         ((isHighlighted || isSelected) && isFullDateRange);

        return isInRange && isVisible;
      });

      if (!foundData) return;

      // Try common link fields
      const url = foundData.url || foundData.link || foundData.href || foundData.permalink;
      if (url && typeof window !== 'undefined') {
        // open in new tab safely
        window.open(url, '_blank', 'noopener,noreferrer');
      }
    }
    
    function handleMouseLeave() {
      hoveredData = lastHoveredData;
      draw();
    }
    
  let resizeObserver;
    onMount(() => {
      ctx = canvas.getContext('2d');
      setupCanvasDPI();
      // Observe size changes for responsiveness / DPR changes
      if (window.ResizeObserver) {
    resizeObserver = new ResizeObserver(() => {
          setupCanvasDPI();
          draw();
        });
    resizeObserver.observe(containerEl);
      }
      // Setup d3-zoom for wheel, double-click, and touch/pinch
  const z = zoom()
        .scaleExtent([MIN_SCALE, MAX_SCALE])
        .filter((event) => {
          // Allow wheel, touch, and dblclick; allow panning with primary button without modifiers
          const e = event;
          if (e.type === 'wheel' || e.type === 'touchstart' || e.type === 'touchmove') return true;
          if (e.type === 'mousedown') return e.button === 0 && !e.ctrlKey && !e.metaKey && !e.shiftKey;
          return !e.ctrlKey && !e.metaKey && !e.shiftKey;
        })
        .on('zoom', (event) => {
          t = event.transform;
          draw();
        });
  select(canvas).call(z);
      draw();
      return () => {
        if (resizeObserver) resizeObserver.disconnect();
      };
    });

  // (Manual pan/zoom handlers removed in favor of d3-zoom)
    
    $: if (ctx) {
      // Redraw when these change and there is data
      data, opacity, selectedValues, searchQuery, showAnnotations, domainColumn, startDate, endDate, innerWidth, innerHeight, xScale, yScale, isFullDateRange;
      if (data.length) draw();
    }
</script>

<div class="chart-container" bind:this={containerEl}>
    <canvas
      bind:this={canvas}
      on:mousemove={handleMouseMove}
      on:mouseleave={handleMouseLeave}
      on:click={handleClick}
    ></canvas>
    
    {#if hoveredData}
      <DetailCard {hoveredData} {data} {domainColumn} {colorScale} posX={tooltipX} posY={tooltipY} />
    {/if}
</div>

<style>
    .chart-container {
      position: relative; /* anchor for absolutely-positioned tooltip */
      display: flex;
      justify-content: center;
      align-items: center;
      flex-direction: column;
      width: 100%;
      height: 100%;
    }
  
    canvas {
      cursor: crosshair;
      border-radius: 8px;
      background-color: white;
      width: 100%;
      height: 100%;
      display: block;
    }
    
</style>