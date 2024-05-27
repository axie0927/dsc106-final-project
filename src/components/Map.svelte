<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let years = Array.from({ length: 2015 - 1975 + 1 }, (_, i) => (1975 + i).toString());
  let selectedCrime = "violent_crimes";
  let violent_crimes = ["violent_crimes","homicides", "rapes", "assaults", "robberies"];
  let crimeData;
  let geojson;
  let selectedYear = "1990";
  let path;
  const width = 800;
  const height = 600;
  let projection;

  onMount(async () => {
    const response = await fetch('us-states.json');
    geojson = await response.json();

    const crime = await d3.csv('crime.csv');
    crimeData = crime.filter(item => item.report_year === selectedYear);

    projection = d3.geoAlbersUsa().translate([width/2, height/2]);
    path = d3.geoPath().projection(projection);

    updateMap();
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

    // Draw map paths
    svg.selectAll('path')
      .data(geojson.features)
      .enter()
      .append('path')
      .attr('d', path)
      .attr('stroke', 'white')
      .attr('fill', '#ddd');

    const bubble = svg.append("g").attr("class", "bubble");

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
      d3.select(this)
        .style("fill", "#589BE5")
        .style("stroke", "#EF4A60")
        .style("opacity", 1);
    };

    const mousemove = function (event, d) {
      const f = d3.format(",");
      tooltip.html("<div style='color: #0072BC'><b>" + d.city + "</b></div><div>Number of " + selectedCrime.replace('_', ' ') + ": " + `${f(d[selectedCrime])}` + "</div>"
      + "<div>Population: " + d.population + "</div>")
        .style("left", (event.x) + 20 + "px")
        .style("top", (event.y + window.pageYOffset) + "px");
    };

    const mouseleave = function () {
      tooltip.style("opacity", 0);
      d3.select(this)
        .style("stroke", "#0072BC")
        .style("opacity", 1);
    };

    // Set bubble scale
    const valueScale = d3.extent(crimeData, d => +d[selectedCrime]);
    const size = d3.scaleSqrt()
      .domain(valueScale)
      .range([1, 40]);

    // Draw bubbles
    bubble.selectAll("circle")
      .data(crimeData)
      .join("circle")
      .attr("cx", d => projection([+d.lng, +d.lat])[0])
      .attr("cy", d => projection([+d.lng, +d.lat])[1])
      .attr("r", d => size(+d[selectedCrime]))
      .style("fill", "#589BE5")
      .attr("stroke", "#0072BC")
      .attr("stroke-width", 0.5)
      .attr("fill-opacity", .6)
      .on("mouseover", mouseover)
      .on("mousemove", mousemove)
      .on("mouseleave", mouseleave)
      .transition() // Add transition
      .duration(500) // Set transition duration in milliseconds
      .attr("r", d => size(+d[selectedCrime])); // Transition to new size;


  }

  function handleYearChange(event) {
    selectedYear = event.target.value;
    d3.csv('crime.csv').then(data => {
      crimeData = data.filter(item => item.report_year === selectedYear);
      updateMap();
    });
  }

  function handleCrimeChange(event) {
    selectedCrime = event.target.value;
    updateMap();
  }
</script>

<div id="container">
  <div id="controls">
    <select bind:value={selectedYear} on:change={handleYearChange}>
      {#each years as year}
        <option value={year}>{year}</option>
      {/each}
    </select>

    <select bind:value={selectedCrime} on:change={handleCrimeChange}>
      {#each violent_crimes as crime}
        <option value={crime}>{crime}</option>
      {/each}
    </select>
  </div>
  <div id="map-container"></div>
</div>

<style>
  #container {
    border: 2px solid black;
    padding: 10px;
    display: flex;
    flex-direction: column;
    align-items: center;
  }
  #controls {
    margin-bottom: 10px;
  }
</style>
