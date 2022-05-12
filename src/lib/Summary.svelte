<script context="module" lang="ts">
  import * as d3 from "d3";

  export type Reading = {
    time: string;
    co2: number;
    temp: number;
    humidity: number;
  };
  export type ParsedReading = {
    time: Date;
    co2: number;
    temp: number;
    humidity: number;
  };

  function C_to_F(deg_in_c: number) {
    return deg_in_c * (9 / 5) + 32;
  }

  function last<T>(arr: T[]) {
    return arr[arr.length - 1];
  }

  function format_time(d: Date, with_day: boolean = false) {
    if (with_day) {
      return d3.timeFormat("%a %b %e @ %I:%M %p")(d);
    }
    return d3.timeFormat("%I:%M %p")(d);
  }

  export type Measurement = "temp" | "co2" | "humidity";

  function parseReadingLine(line: string): ParsedReading {
    const [timestamp, co2, temp_in_c, humidity] = line.split(",").map(Number);

    if (!(timestamp && co2 && temp_in_c && humidity)) {
      console.error("Can't parse line of data", line);
      throw new Error("Can't parse line of data");
    }

    return {
      time: new Date(timestamp * 1000),
      co2: co2,
      temp: C_to_F(Number(temp_in_c)),
      humidity: humidity,
    };
  }

  const get_observations = async (num_hours: number) => {
    const req = await fetch(`https://wmadisonatelier.com/data/${num_hours}`);

    const parsed_readings: ParsedReading[] = (await req.json()).map(
      parseReadingLine
    );
    return parsed_readings;
  };
</script>

<script lang="ts">
  import SummaryChart from "./SummaryChart.svelte";

  let num_hours = 3;

  // Pull data from Pie server and update the air_readings variable
  $: obs_req = get_observations(num_hours);
</script>

<label>
  <span>Number of hours to pull: </span>
  <input type="number" bind:value={num_hours} min="1" max="24" />
</label>

{#await obs_req}
  <p>Fetching data from office raspberry pi...</p>
{:then readings}
  <h3>
    Time range: {format_time(readings[0].time, true)} - {format_time(
      last(readings).time
    )}
  </h3>
  <SummaryChart data={readings} name="co2" unit="ppm" />
  <SummaryChart data={readings} name="temp" unit="degrees" />
  <SummaryChart data={readings} name="humidity" unit="%" />
{:catch error}
  <p>Something went wrong: {error.message}</p>
{/await}

<style>
  label {
    display: flex;
    justify-content: center;
    align-items: center;
    gap: 5px;
  }
</style>
