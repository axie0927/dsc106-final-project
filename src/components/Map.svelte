<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let years = Array.from({ length: 2014 - 1975 + 1 }, (_, i) => (1975 + i).toString());
  let selectedCrime = "violent_crimes";
  let violent_crimes = ["violent_crimes", "homicides", "rapes", "assaults", "robberies"];
  let crimeData;
  let geojson;
  let selectedYear = "1990";
  let path;
  const width = 1200;
  const height = 600;
  const linePlotWidth = 700;
  const linePlotHeight = 600;
  let projection;
  let selected = null;
  let previouslySelectedBubble = null;  
  let crime;

  onMount(async () => {
    const response = await fetch('us-states.json');
    geojson = await response.json();

    crime = await d3.csv('crime.csv');
    crimeData = crime.filter(item => item.report_year === selectedYear);

    projection = d3.geoAlbersUsa().translate([width / 2, height / 2]);
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
      if (selected !== d.city) {
        d3.select(this)
          .style("fill", "#589BE5")
          .style("stroke", "#EF4A60")
          .style("opacity", 1);
      }
    };

    const mousemove = function (event, d) {
      const f = d3.format(",");
      tooltip.html("<div style='color: #0072BC'><b>" + d.city + "</b></div><div>Number of " + selectedCrime.replace('_', ' ') + ": " + `${f(d[selectedCrime])}` + "</div>"
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
    const valueScale = d3.extent(crimeData, d => +d[selectedCrime]);
    const size = d3.scaleSqrt()
      .domain(valueScale)
      .range([1, 50]);

    // Draw bubbles
    bubble.selectAll("circle")
      .data(crimeData)
      .join("circle")
      .attr("cx", d => projection([+d.lng, +d.lat])[0])
      .attr("cy", d => projection([+d.lng, +d.lat])[1])
      .attr("r", d => size(+d[selectedCrime]))
      .style("fill", d => selected === d.city ? "orange" : "#589BE5") // Set color based on selection
      .attr("stroke", "#0072BC")
      .attr("stroke-width", 0.5)
      .attr("fill-opacity", .6)
      .on("mouseover", mouseover)
      .on("mousemove", mousemove)
      .on("mouseleave", mouseleave)
      .on("click", handleClick) 
  }

  function createLinePlot() {
    // Clear previous line plot
    d3.select('#line-plot-container').selectAll('svg').remove();

    var crime_city = crime.filter(item => item.city === selected)
    var svgs = d3.select('#line-plot-container').append('svg')
      .attr('width', linePlotWidth)
      .attr('height', linePlotHeight)
      .style('border', '1px solid black'); // Add border to differentiate the line plot

    // Set the ranges
    var xScale = d3.scaleTime().range([40, linePlotWidth - 20]);
    var yScaleCrime = d3.scaleLinear().range([linePlotHeight - 30, 20]);
    var yScaleUnemployment = d3.scaleLinear().range([linePlotHeight - 30, 20]);

    // Define the line functions
    var crimeLine = d3.line()
      .x(d => xScale(new Date(d.report_year)))
      .y(d => yScaleCrime(+d[selectedCrime]));

    var unemploymentLine = d3.line()
      .x(d => xScale(new Date(d.report_year)))
      .y(d => yScaleUnemployment(+d.unemploymentRate));

    // Scale the range of the data
    xScale.domain(d3.extent(crime_city, d => new Date(d.report_year)));
    yScaleCrime.domain([0, d3.max(crime_city, d => +d[selectedCrime])]);
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
    var legend = svgs.append("g")
      .attr("transform", `translate(${linePlotWidth / 2 - 100}, ${linePlotHeight - 50})`);

    legend.append("circle")
      .attr("cx", 0)
      .attr("cy", 0 )
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
    createLinePlot();
  }
</script>

<div id="container">
  <div id="controls">
    <select bind:value={selectedYear} on:change={handleYearChange      }>
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
  <div id="line-plot-container"></div> <!-- Added container for the line plot -->
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
  #map-container, #line-plot-container {
    margin-top: 20px;
  }
</style>

