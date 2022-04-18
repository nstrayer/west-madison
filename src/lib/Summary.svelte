<script context="module" lang="ts">
  import * as d3 from "d3";

  export type Reading = {
    time: string;
    co2: number;
    temp: number;
    humidity: number;
  };

  function C_to_F(deg_in_c: number) {
    return deg_in_c * (9 / 5) + 32;
  }

  export type Measurement = "temp" | "co2" | "humidity";

  const parseTime = d3.timeParse("%m/%d/%y %H:%M:%S");
</script>

<script lang="ts">
  import SummaryChart from "./SummaryChart.svelte";

  const num_lines = 1000;

  // Pull data from Pie server and update the air_readings variable

  const get_observations = async () => {
    const req = await fetch(`http://67.205.131.141/data?nlines=${num_lines}`);
    const readings = (await req.json()).readings;

    return readings
      .map((d) => {
        const time = parseTime(d.time);

        if (time === null) return null;

        const temp = C_to_F(d.temp);
        const humidity = d.humidity;
        const co2 = d.co2;

        return {
          time,
          temp,
          humidity,
          co2,
        };
      })
      .filter((d) => d !== null);
  };

  const obs_req = get_observations();
</script>

{#await obs_req}
  <p>Fetching data from office raspberry pi...</p>
{:then readings}
  <SummaryChart data={readings} name="co2" unit="ppm" />
  <SummaryChart data={readings} name="temp" unit="degrees" />
  <SummaryChart data={readings} name="humidity" unit="%" />
{:catch error}
  <p>Something went wrong: {error.message}</p>
{/await}
