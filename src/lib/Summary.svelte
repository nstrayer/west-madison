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

  const parseTime = d3.timeParse("%m/%d/%y %H:%M:%S");
</script>

<script lang="ts">
  import SummaryChart from "./SummaryChart.svelte";

  const num_lines = 7000;

  // Pull data from Pie server and update the air_readings variable

  const get_observations = async () => {
    const req = await fetch(
      `https://wmadisonatelier.com/data?nlines=${num_lines}`
    );
    const readings = (await req.json()).readings as Reading[];

    return readings
      .map((d) => {
        const time = parseTime(d.time);

        if (time === null) return null;

        // Move the time to the correct time zone in the least robust way possible
        time.setHours(time.getHours() - 5);

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
