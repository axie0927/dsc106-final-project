<script>
  import Map from './Map.svelte';
</script>

<main>
  <h1>Number of Violent Crimes in the USA and Comparison with Employment Rate</h1>
  <p>Select specific year or type of crime to see the number of crimes for specific cities in the US during the selected year. A more in depth visualization between number of crimes and unemployment rate can be seen below the map.</p>
  <p>Hover over the circles below to see the number of crimes as well as the population of the city!</p>
  <Map />

</main>

<style>
  main {
    text-align: center;
    padding: 1em;
    max-width: 800px;
    margin: 0 auto;
  }
  h1 {
    color: #333;
  }
</style>