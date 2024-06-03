<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let years = Array.from({ length: 2014 - 1975 + 1 }, (_, i) => (1975 + i).toString());
  let selectedCrime = "Violent Crimes";
  let violent_crimes = ["Violent Crimes", "Homicides", "Rapes", "Assaults", "Robberies"];
  let violent_crimes_dict = {"Violent Crimes" :"violent_crimes", "Homicides": "homicides","Rapes": "rapes", "Assaults": "assaults","Robberies": "robberies"};
  let selectedCrimeIndex = violent_crimes_dict[selectedCrime];
  let crimeData;
  let geojson;
  let selectedYear = "1975";
  let path;
  const width = 1200;
  const height = 600;
  const linePlotWidth = 700;
  const linePlotHeight = 600;
  let projection;
  let selected = null;
  let previouslySelectedBubble = null;  
  let crime;
  let corrData;
  let curCorr;
  

  let newZoom;

  onMount(async () => {
    const response = await fetch('us-states.json');
    geojson = await response.json();

    crime = await d3.csv('crime.csv');
    crimeData = crime.filter(item => item.report_year === selectedYear);

    projection = d3.geoAlbersUsa().translate([width / 2, height / 2]);
    path = d3.geoPath().projection(projection);

    const corrResponse = await fetch('city_correlation.json');
    const corrJson = await corrResponse.json();
    corrData = corrJson; 
    curCorr = corrJson[selectedCrimeIndex]

    updateMap();
    createHeatmap();
  });

  function updateMap() {
  // Check if SVG exists, otherwise create it
  let svg = d3.select('#map-container').select('svg');
  if (svg.empty()) {
    svg = d3.select('#map-container').append('svg')
      .attr('width', width)
      .attr('height', height);
  }

  // Clear previous map paths, bubbles, tooltips, and legends
  svg.selectAll('path').remove();
  svg.selectAll('circle').remove();
  svg.selectAll('g.bubble').remove();
  d3.select('body').selectAll('.tooltip').remove();
  svg.selectAll('g.legend').remove();

  // put bubbles and geomap into a group
  let mapGroup = svg.select('g.map-content');
  if (mapGroup.empty()) {
    mapGroup = svg.append('g').attr('class', 'map-content');
  }


  // Draw map paths
  mapGroup.selectAll('path')
    .data(geojson.features)
    .enter()
    .append('path')
    .attr('d', path)
    .attr('stroke', 'white')
    .attr('fill', '#ddd'); // Set to none since we are not using heatmap colors here

  const bubble = mapGroup.append("g").attr("class", "bubble");

  // Create a tooltip
  const tooltip = d3.select('body')
    .append('div')
    .attr('class', 'tooltip')
    .style('opacity', 0)
    .style("background-color", "white")
    .style("border", "solid")
    .style("border-width", "2px")
    .style("border-radius", "5px")
    .style("padding", "5px")
    .style("position", "absolute");

  // Tooltip and mouse events
  const mouseover = function (event, d) {
    tooltip.style("opacity", 1);
    if (selected !== d.city) {
      d3.select(this)
        .style("fill", "#589BE5")
        .style("stroke", "#EF4A60")
        .style("opacity", 1);
    }
  };

  const mousemove = function (event, d) {
    const f = d3.format(",");
    tooltip.html("<div style='color: #0072BC'><b>" + d.city + `</b></div><div>Number of ${selectedCrime}: ${f(d[selectedCrimeIndex])}</div>`
      + "<div>Population: " + d.population + "</div>")
      .style("left", (event.x) + 20 + "px")
      .style("top", (event.y + window.scrollY) + "px");
  };

  const mouseleave = function (event, d) {
    tooltip.style("opacity", 0);
    if (selected !== d.city) {
      d3.select(this)
        .style("fill", "#589BE5")
        .style("stroke", "#0072BC")
        .style("opacity", 1);
    }
  };

  // Click event to select bubble
  const handleClick = function (event, d) {
    if (previouslySelectedBubble) {
      previouslySelectedBubble.style("fill", "#589BE5"); // Reset the color of the previously selected bubble
    }
    d3.select(this).style("fill", "orange"); // Highlight selected bubble
    selected = d.city; // Save selected city
    previouslySelectedBubble = d3.select(this); // Save the reference to the currently selected bubble
    createLinePlot();
  };

  // Set bubble scale
  var valueScale = [0, 50000];
  var legendValues = [2500, 5000, 10000];
  if (selectedCrime === "Rapes") {
    valueScale = [0, 5000]
    legendValues = [400, 800, 1600];
  } else if (selectedCrime === "Homicides") {
    valueScale = [0, 2000]
    legendValues = [100, 200, 400];
  }
  const size = d3.scaleSqrt()
    .domain(valueScale)
    .range([1, 50]);

  // Draw bubbles
  bubble.selectAll("circle")
    .data(crimeData)
    .join("circle")
    .attr("cx", d => projection([+d.lng, +d.lat])[0])
    .attr("cy", d => projection([+d.lng, +d.lat])[1])
    .attr("r", d => size(+d[selectedCrimeIndex]))
    .style("fill", d => selected === d.city ? "orange" : "#589BE5") // Set color based on selection
    .attr("stroke", "black")
    .attr("stroke-width", 0.8)
    .attr("fill-opacity", .6)
    .on("mouseover", mouseover)
    .on("mousemove", mousemove)
    .on("mouseleave", mouseleave)
    .on("click", handleClick)

  // Add Legend
  const legendSize = legendValues.map(d => size(d));

  const legend = svg.append("g")
    .attr("class", "legend")
    .attr("transform", "translate(50,50)");

  legend.selectAll("circle")
    .data(legendValues)
    .enter()
    .append("circle")
    .attr("cx", 0)
    .attr("cy", (d, i) => i * 50)
    .attr("r", (d, i) => legendSize[i])
    .style("fill", "#589BE5")
    .style("opacity", 0.6);

  legend.selectAll("text")
    .data(legendValues)
    .enter()
    .append("text")
    .attr("x", 40)
    .attr("y", (d, i) => i * 50)
    .attr("dy", "0.35em")
    .text(d => `${Math.round(d)}  ${selectedCrime}`)
    .style("font-size", "12px")
    .style("fill", "#333");

  svg.call(d3.drag()
    .on("start", dragStart)
    .on("drag", drag)
    .on("end", dragEnd)
  );
  let prevX = 0;
  let prevY = 0;
  let zoomValue = 1;
  let currentX = 0;
  let currentY = 0;
  function dragStart(event) {
    prevX = event.x;
    prevY = event.y;
    mapGroup.attr('cursor', 'grabbing');
  } 

  function drag(event) {
    const dx = event.x - prevX;
    const dy = event.y - prevY;
    zoomValue = newZoom; 
    currentX += dx;
    currentY += dy;

    mapGroup.attr('transform', `translate(${currentX},${currentY}) scale(${zoomValue})`);

    prevX = event.x;
    prevY = event.y;
  }

  function dragEnd(event) {
    mapGroup.attr('cursor', 'grab');

  }
}


  function createHeatmap() {
    const heatmapWidth = 600;
    const heatmapHeight = 400;
    const margin = { top: 30, right: 20, bottom: 40, left: 40 };
    const cellSize = 50;
    d3.select('#heatmap-container').selectAll('svg').remove();
    const svg = d3.select("#heatmap-container")
      .append("svg")
      .attr("width", heatmapWidth + margin.left + margin.right)
      .attr("height", heatmapHeight + margin.top + margin.bottom)
      .append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

    const cities = Object.keys(curCorr);
    const numCols = Math.ceil(Math.sqrt(cities.length));
    const numRows = Math.ceil(cities.length / numCols);

    const colorScale = d3.scaleSequential(d3.interpolateRdYlGn)
      .domain([-1, 1]);

    const xScale = d3.scaleBand()
      .range([0, heatmapWidth])
      .domain(d3.range(numCols));

    const yScale = d3.scaleBand()
      .range([0, heatmapHeight])
      .domain(d3.range(numRows));

    svg.selectAll("rect")
      .data(cities)
      .enter()
      .append("rect")
      .attr("x", (d, i) => xScale(i % numCols))
      .attr("y", (d, i) => yScale(Math.floor(i / numCols)))
      .attr("width", cellSize + 30)
      .attr("height", cellSize)
      .style("fill", d => colorScale(curCorr[d]));

    svg.selectAll("text")
      .data(cities)
      .enter()
      .append("text")
      .attr("x", (d, i) => xScale(i % numCols) + cellSize / 2)
      .attr("y", (d, i) => yScale(Math.floor(i / numCols)) + cellSize / 2)
      .attr("dy", "2em")
      .attr("text-anchor", "middle")
      .style("font-size", "8px")
      .text(d => d);

    svg.append("text")
      .attr("x", heatmapWidth / 2)
      .attr("y", -10)
      .attr("text-anchor", "middle")
      .style("font-size", "16px")
      .text(`Correlation between Unemployment and ${selectedCrime}`);

    const legend = svg.append("g")
      .attr("class", "legend")
      .attr("transform", `translate(0,${heatmapHeight + 10})`);

    const legendScale = d3.scaleLinear()
      .range([0, heatmapWidth])
      .domain([-1, 1]);

    const legendAxis = d3.axisBottom(legendScale)
      .ticks(5);

    legend.selectAll("rect")
      .data(d3.range(-1, 1.01, 0.01))
      .enter()
      .append("rect")
      .attr("x", d => legendScale(d))
      .attr("y", 0)
      .attr("width", heatmapWidth / 200)
      .attr("height", 10)
      .style("fill", d => colorScale(d));

    legend.append("g")
      .attr("transform", `translate(0,10)`)
      .call(legendAxis);
  }

  function handleYearChange(event) {
    selectedYear = event.target.value;
    crimeData = crime.filter(item => item.report_year === selectedYear);
    updateMap();
  }

  function handleCrimeChange(event) {
    selectedCrime = event.target.value;
    selectedCrimeIndex = violent_crimes_dict[selectedCrime];
    curCorr = corrData[selectedCrimeIndex]
    updateMap();
    createHeatmap()
  }
  function handleZoomChange(event) {
    const zoomValue = event.target.value / 50; 
    const svg = d3.select('#map-container').select('svg');
    const mapContent = svg.select('g.map-content');
    mapContent.attr('transform', `scale(${zoomValue})`);
    newZoom = zoomValue;
  }

  function createLinePlot() {
    if (!selected) return;

    const crime_city = crime.filter(item => item.city === selected);

    // Remove previous line plot if it exists
    d3.select('#line-plot-container').selectAll('svg').remove();

    const svgs = d3.select('#line-plot-container')
      .append('svg')
      .attr('width', linePlotWidth)
      .attr('height', linePlotHeight)
      .style('border', '1px solid black');

    // Set the ranges
    const xScale = d3.scaleTime().range([40, linePlotWidth - 20]);
    const yScaleCrime = d3.scaleLinear().range([linePlotHeight - 30, 20]);
    const yScaleUnemployment = d3.scaleLinear().range([linePlotHeight - 30, 20]);

    // Define the line functions
    const crimeLine = d3.line()
      .x(d => xScale(new Date(d.report_year)))
      .y(d => yScaleCrime(+d[selectedCrimeIndex]));

    const unemploymentLine = d3.line()
      .x(d => xScale(new Date(d.report_year)))
      .y(d => yScaleUnemployment(+d.unemploymentRate));

    // Scale the range of the data
    xScale.domain(d3.extent(crime_city, d => new Date(d.report_year)));
    yScaleCrime.domain([0, d3.max(crime_city, d => +d[selectedCrimeIndex])]);
    yScaleUnemployment.domain([0, d3.max(crime_city, d => +d.unemploymentRate)]);

    // Add the crime path
    svgs.append("path")
      .datum(crime_city)
      .attr("class", "line")
      .attr("d", crimeLine)
      .style("stroke", "blue")
      .style("fill", "none");

    // Add the unemployment rate path
    svgs.append("path")
      .datum(crime_city)
      .attr("class", "line")
      .attr("d", unemploymentLine)
      .style("stroke", "red")
      .style("fill", "none");

    // Add the X Axis
    svgs.append("g")
      .attr("transform", `translate(0,${linePlotHeight - 30})`)
      .call(d3.axisBottom(xScale));

    // Add the Crime Y Axis
    svgs.append("g")
      .attr("transform", "translate(40,0)")
      .call(d3.axisLeft(yScaleCrime).ticks(5))
      .style("color", "blue");

    // Add the Unemployment Rate Y Axis
    svgs.append("g")
      .attr("transform", `translate(${linePlotWidth - 20},0)`)
      .call(d3.axisRight(yScaleUnemployment).ticks(5))
      .style("color", "red");

    // Add chart title
    svgs.append("text")
      .attr("x", linePlotWidth / 2)
      .attr("y", 20)
      .attr("text-anchor", "middle")
      .style("font-size", "16px")
      .style("text-decoration", "underline")
      .text("Crime and Unemployment Rate Over Time");

    // Add legend
    const legend = svgs.append("g")
      .attr("transform", `translate(${linePlotWidth / 2 - 100}, ${linePlotHeight - 50})`);

    legend.append("circle")
      .attr("cx", 0)
      .attr("cy", 0)
      .attr("r", 6)
      .style("fill", "blue");

    legend.append("text")
      .attr("x", 15)
      .attr("y", 0)
      .text(selectedCrime)
      .style("font-size", "12px")
      .attr("alignment-baseline", "middle");

    legend.append("circle")
      .attr("cx", 110)
      .attr("cy", 0)
      .attr("r", 6)
      .style("fill", "red");

    legend.append("text")
      .attr("x", 125)
      .attr("y", 0)
      .text("Unemployment Rate")
      .style("font-size", "12px")
      .attr("alignment-baseline", "middle");
  }
  
</script>

<div id="container">
  <div id="controls" class="horizontal-controls">
    <div class="control">
      <label for="year-selector">Year:</label>
      <select id="year-selector" bind:value={selectedYear} on:change={handleYearChange}>
        {#each years as year}
          <option value={year}>{year}</option>
        {/each}
      </select>
    </div>

    <div class="control">
      <label for="crime-selector">Category:</label>
      <select id="crime-selector" bind:value={selectedCrime} on:change={handleCrimeChange}>
        {#each violent_crimes as crime}
          <option value={crime}>{crime}</option>
        {/each}
      </select>
    </div>

    <div class="control">
      <label for="zoom-slider">Zoom:</label>
      <input type="range" id="zoom-slider" min="1" max="100" value="50" on:input={handleZoomChange} />
    </div>
  </div>
  <div id="map-container"></div>
  <p>Clicking onto a bubble will show a line graph of number of crimes as well as unemployment rate bewteen the years 1975 and 2014.</p>
  <div id="line-plot-container"></div>
  <p>Below is a heat map showing the overall correlation between number of crimes for the selected crime and unemployment rates for each city listed on the map above.</p>
  <div id="heatmap-container"></div>
</div>

<style>
  #container {  
    display: flex;
    flex-direction: column;
    align-items: center;
  }
  
  .horizontal-controls {
    display: flex;
    justify-content: space-between;
    width: 80%;
    max-width: 800px;
    margin-bottom: 20px;
  }

  .control {
    flex: 1;
    margin-bottom: 10px;
  }

  .control label {
    display: block;
    margin-bottom: 5px;
  }

  #map-container, #line-plot-container, #heatmap-container {
    margin-top: 20px;
  }
</style>
