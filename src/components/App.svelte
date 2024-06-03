<script>
  import Map from './Map.svelte';
</script>

<main>
  <h1>Number of Violent Crimes in the USA and Comparison with Employment Rate</h1>
  <h4>Select specific year or type of crime to see the number of crimes for specific cities in the US during the selected year. A more in depth visualization between number of crimes and unemployment rate can be seen below the map.</h4>
  <div class="map-container">
    <Map />
  </div>

</main>

<style>
  html, body {
    height: 100%;
    margin: 0;
  }
  main {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    height: 100%;
    text-align: center;
    padding: 1em;
    max-width: 800px;
    margin: 0 auto;
  }
  h1 {
    color: #333;
    text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.3);
    padding: 0.5em;
    border-bottom: 2px solid #333;
    margin-bottom: 1em;
  }
  p {
    margin: 1em 0;
  }
  .map-container {
    border: 2px solid black;
    display: inline-block;
    padding: 1em; 
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  }
</style>