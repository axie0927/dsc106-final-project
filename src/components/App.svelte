<script>
  import Map from './Map.svelte';
</script>

<main>
  <h1>Crimes in the USA</h1>
  <p>For our visualizations, we have a map of the US showing the total number of the selected crimes that occurred during the selected year, with the circle sizes being proportional to the numbers. If you hover over each circle, it will show the exact number of crimes as well as the population of that specific city. Additionally, if you click on the bubbles, you will see a layered line graph that shows the number of crimes and the employment rate over the years 1975 to 2014.</p>
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