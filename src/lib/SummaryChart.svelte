<script context="module" lang="ts">
  function roundTo(x: number, precision: number) {
    const multiplier = Math.pow(10, precision);

    return Math.round(x * multiplier) / multiplier;
  }

  function avgVal(arr: number[], precision: number = 1) {
    const n_obs = arr.length;
    return roundTo(
      arr.reduce((s, x) => s + x / n_obs, 0),
      1
    );
  }

  type Observation = { val: number; time: Date };

  const margin_top = 20; // top margin, in pixels
  const margin_right = 30; // right margin, in pixels
  const margin_bottom = 30; // bottom margin, in pixels
  const margin_left = 40;

  const extents_by_measure: Record<Measurement, [number, number]> = {
    temp: [60, 80],
    co2: [300, 1000],
    humidity: [0, 80],
  };

  const curve = d3.curveLinear;
</script>

<script lang="ts">
  import * as d3 from "d3";

  import { onMount } from "svelte";
  import type { Measurement, ParsedReading } from "./Summary.svelte";

  let el: HTMLDivElement;
  export let unit: string;
  export let data: ParsedReading[];
  export let name: Measurement;

  const values: Observation[] = data.map((d) => ({
    val: d[name],
    time: d.time,
  }));
  const n_obs = values.length;

  const average_value = avgVal(values.map((x) => x.val));

  const current_value = avgVal(values.slice(-10).map((x) => x.val));

  const [y_min, y_max] = d3.extent(data, (d) => d[name]);
  const y_domain = extents_by_measure[name];
  if (y_domain[0] > y_min) y_domain[0] = y_min;
  if (y_domain[1] < y_max) y_domain[1] = y_max;

  const x_domain = d3.extent(data, (d) => d.time);

  const I = d3.range(n_obs);

  onMount(() => {
    const chart_w = el.offsetWidth;
    const chart_h = el.offsetHeight;

    const x_scale = d3
      .scaleTime(x_domain, [margin_left, chart_w - margin_right])
      .nice();

    const y_scale = d3.scaleLinear(y_domain, [
      chart_h - margin_bottom,
      margin_top,
    ]);

    const x_axis = d3.axisBottom(x_scale).ticks(8).tickSizeOuter(0);
    const y_axis = d3.axisLeft(y_scale).ticks(chart_h / 40);

    // Construct a line generator.
    const line = d3
      .line<Observation>()
      .curve(curve)
      .x((d: Observation) => x_scale(d.time))
      .y((d: Observation) => y_scale(d.val));

    const svg = d3
      .create("svg")
      .attr("width", chart_w)
      .attr("height", chart_h)
      .attr("viewBox", [0, 0, chart_w, chart_h])
      .attr("style", "max-width: 100%; height: auto; height: intrinsic;");

    svg
      .append("g")
      .attr("transform", `translate(0,${chart_h - margin_bottom})`)
      .call(x_axis);

    svg
      .append("g")
      .attr("transform", `translate(${margin_left},0)`)
      .call(y_axis)
      .call((g) => g.select(".domain").remove())
      .call((g) =>
        g
          .selectAll(".tick line")
          .clone()
          .attr("x2", chart_w - margin_left - margin_right)
          .attr("stroke-opacity", 0.1)
      )
      .call((g) =>
        g
          .append("text")
          .attr("x", -margin_left)
          .attr("y", 10)
          .attr("fill", "currentColor")
          .attr("text-anchor", "start")
          .text(unit)
      );

    svg
      .append("path")
      .attr("fill", "none")
      .attr("stroke", "currentColor")
      .attr("stroke-width", 1.5)
      .attr("stroke-opacity", 0.65)
      .attr("d", line(values));

    el.append(svg.node());
  });
</script>

<div class="container">
  <h2>
    {name} = {current_value}
    <span class="units">{unit}</span>
  </h2>
  <p>
    avg: {average_value} <span class="units">{unit}</span>
  </p>
  <div class="line-chart" bind:this={el} />
</div>

<style>
  .container {
    --shadow-color: 0deg 0% 63%;
    --shadow-elevation-low: 0.2px 0.4px 0.5px hsl(var(--shadow-color) / 0.34),
      0.3px 0.7px 0.9px -1.2px hsl(var(--shadow-color) / 0.34),
      0.7px 1.6px 2px -2.5px hsl(var(--shadow-color) / 0.34);
    --shadow-elevation-medium: 0.2px 0.4px 0.5px hsl(var(--shadow-color) / 0.36),
      0.6px 1.3px 1.6px -0.8px hsl(var(--shadow-color) / 0.36),
      1.5px 3.4px 4.2px -1.7px hsl(var(--shadow-color) / 0.36),
      3.6px 8.2px 10.1px -2.5px hsl(var(--shadow-color) / 0.36);
    /* outline: 1px solid salmon; */

    margin-inline: auto;
    margin-block: 15px;
    box-shadow: var(--shadow-elevation-medium);
    background-color: hsl(180 50% 93%);
    color: hsl(248 31% 15%);
    width: min(500px, 100%);
    padding: 15px;

    border-radius: 1px;
  }

  h2,
  p {
    font-weight: 200;
    text-align: right;
    margin: 0;
  }

  h2 {
    margin-bottom: 5px;
    font-size: 1.6rem;
  }
  p {
    margin-bottom: 10px;
  }

  .units {
    opacity: 0.75;
    font-size: 0.8rem;
  }

  .line-chart {
    height: 150px;
  }
</style>
