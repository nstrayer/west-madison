<script lang="ts">
  import logo from "./assets/madison-house.png";
  import Summary from "./lib/Summary.svelte";
</script>

<main>
  <img src={logo} alt="Svelte Logo" />
  <h1>West Madison HomeBase</h1>
  <Summary />
</main>

<style>
  :root {
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen,
      Ubuntu, Cantarell, "Open Sans", "Helvetica Neue", sans-serif;
    background-color: #fffffe;
    box-sizing: border-box;
  }

  :global(*, *::after, *::before) {
    box-sizing: border-box;
  }
  main {
    text-align: center;
    padding: 1em;
    margin: 0 auto;
  }

  img {
    height: 16rem;
    width: 16rem;
  }

  h1 {
    color: #fab91d;
    font-weight: 150;
    line-height: 1.1;
    text-transform: uppercase;
    font-size: 4rem;
    margin-block: -5px 30px;
    margin-inline: auto;
    max-width: 50ch;
  }

  @media (min-width: 480px) {
    h1 {
      max-width: none;
    }
  }
</style>
