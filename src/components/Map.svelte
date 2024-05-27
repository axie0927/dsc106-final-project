<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let years = Array.from({ length: 2015 - 1975 + 1 }, (_, i) => 1975 + i);
  let violent_crimes = ["homicides_percapita", "rapes_percapita", "assaults_percapita", "robberies_percapita"];
  let selectedYear = 1975;
  let selectedCrime = "homicides_percapita";
  let geojson;
  let dataTotal;
  let pointsData;
  let crimeData;

  onMount(async () => {
    const response_map = await fetch('us-states.json');
    geojson = await response_map.json();

    const response_data = await fetch('state-total.json');  
    const data_total = await response_data.json();
    dataTotal = data_total;

    const response_points = await fetch('cities.geojson');
    const pointsText = await response_points.json();
    pointsData = pointsText.features.filter(feature => feature.geometry.type === 'Point');

    const response_crime = await fetch('crimes.json');
    const crime_stats = await response_crime.json();
    crimeData = crime_stats;

    updateMap();

  });

  function updateMap() {
    
    const width = 800;
    const height = 600;

    const svg = d3.select('svg')
      .attr('width', width)
      .attr('height', height);

    // Remove previous paths
    svg.selectAll('path').remove();

    const projection = d3.geoAlbersUsa()
      .scale(1)
      .translate([0, 0]);

    const path = d3.geoPath().projection(projection);

    // Fix size of map
    const bounds = path.bounds(geojson),
          scale = 0.95 / Math.max((bounds[1][0] - bounds[0][0]) / width, (bounds[1][1] - bounds[0][1]) / height),
          translate = [(width - scale * (bounds[1][0] + bounds[0][0])) / 2, (height - scale * (bounds[1][1] + bounds[0][1])) / 2];

    projection
      .scale(scale)
      .translate(translate);

    // Set color scale
    const populationValues = Object.values(dataTotal[selectedYear]).map(stateData => stateData.violent_crimes);
    const minPopulation = d3.min(populationValues);
    const medianPopulation = d3.median(populationValues);
    const maxPopulation = d3.max(populationValues);

    const colorScale = d3.scaleLinear()
                        .domain([minPopulation, medianPopulation, maxPopulation])
                        .range(["lightblue", "blue", "darkblue"]);

    // Set up tooltip for data points
    const Tooltip = d3.select('body')
                  .append('div')
                  .attr('class', 'tooltip')
                  .style('opacity', 0)
                  .attr("class", "tooltip")
                  .style("background-color", "white")
                  .style("border", "solid")
                  .style("border-width", "2px")
                  .style("border-radius", "5px")
                  .style("padding", "5px")
                  .style("position", "absolute");

    const mouseover1 = function(event, d) {
        Tooltip.style('opacity', 1);
        d3.select(this)
          .style('stroke', 'blue')
          .style('opacity', 1);
    };

    const mouseover2 = function(event, d) {
        Tooltip.style('opacity', 1);
        d3.select(this)
          .style('stroke', 'black')
          .style('opacity', 1);
    };

    const mousemove1 = function(event, d) {
        const state = d.properties.name;
        const num_crimes = dataTotal[selectedYear][state].violent_crimes
        Tooltip.html(`State: ${state}<br>Total Number of Crimes: ${num_crimes}`)
            .style('left', (event.x) + 20 + "px")
            .style('top', (event.y + window.pageYOffset) + "px");
    };

    const mousemove2 = function(event, d) {
        const city = d.properties.name;
        const tempCrimeData = crimeData[selectedYear][selectedCrime][city]
        const population = tempCrimeData.population
        const crimeValue = tempCrimeData[selectedCrime]
        Tooltip.html(`City: ${city}<br>City Population: ${population}<br>Number of Crimes per Capita: ${crimeValue}<br>`)
            .style('left', (event.x) + 20 + "px")
            .style('top', (event.y + window.pageYOffset) + "px");
    };

    const mouseleave1 = function(event, d) {
        Tooltip.style('opacity', 0);
        d3.select(this)
          .style('stroke', 'none')
          .style('opacity', 0.8);
    };

    const mouseleave2 = function(event, d) {
        Tooltip.style('opacity', 0);
        d3.select(this)
          .style('stroke', 'none')
          .style('opacity', 0.8);
    };

    svg.selectAll('path')
      .data(geojson.features)
      .enter().append('path')
      .attr('d', path)
      .attr('fill', d => {
        const state = d.properties.name;
        const yearData = dataTotal[selectedYear];
        if (yearData && yearData[state] && yearData[state].violent_crimes) {
          const total = yearData[state].violent_crimes;
          return colorScale(total);
        }
        return 'grey';
      })
      .attr('stroke', 'white')
      .on('mouseover', mouseover1)
      .on('mousemove', mousemove1)
      .on('mouseleave', mouseleave1);

    // Filter pointsData to include only cities with data
    const citiesWithData = Object.keys(crimeData[selectedYear][selectedCrime]);

    const filteredPointsData = pointsData.filter(feature => {
        const city = feature.properties.name;
        return citiesWithData.includes(city);
    });

    svg.append('g')
      .attr('class', 'points')
      .selectAll('circle')
      .data(filteredPointsData)
      .enter().append('circle')
      .attr('cx', d => projection(d.geometry.coordinates)[0])
      .attr('cy', d => projection(d.geometry.coordinates)[1])
      .attr('r', 5)
      .style('fill', 'red')
      .on('mouseover', mouseover2)
      .on('mousemove', mousemove2)
      .on('mouseleave', mouseleave2);
  }

  function handleYearChange(event) {
    selectedYear = event.target.value;
    updateMap();
  }

  function handleCrimeChange(event) {
    selectedCrime = event.target.value;
    updateMap();
  }
  
</script>

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

<svg></svg>

<style>
  svg {
    border: 1px solid black;
  }
</style>