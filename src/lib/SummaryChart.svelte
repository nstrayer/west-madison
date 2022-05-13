<script context="module" lang="ts">
  function avgVal(arr: number[], precision: number = 1) {
    const n_obs = arr.length;
    return arr.reduce((s, x) => s + x / n_obs, 0);
  }

  type Observation = { val: number; time: Date };

  const extents_by_measure: Record<Measurement, [number, number]> = {
    temp: [60, 80],
    co2: [300, 1000],
    humidity: [0, 80],
  };

  const curve = d3.curveLinear;

  const format_num = d3.format(",.1~f");
</script>

<script lang="ts">
  import * as d3 from "d3";
  import { onMount } from "svelte";
  import type { Measurement, ParsedReading } from "./Summary.svelte";

  const margin_top = 24; // top margin, in pixels
  const margin_right = 20; // right margin, in pixels
  const margin_bottom = 22; // bottom margin, in pixels
  const margin_left = 50;
  const container_pad = 18; // How much padding does the plot have around it?

  let el: HTMLDivElement;
  let min_annotation: HTMLSpanElement;
  let max_annotation: HTMLSpanElement;

  export let unit: string;
  export let data: ParsedReading[];
  export let name: Measurement;

  const observations: Observation[] = data.map((d) => ({
    val: d[name],
    time: d.time,
  }));

  const values = observations.map((x) => x.val);
  const current_value = format_num(avgVal(values.slice(-10)));
  const min_obs = observations[d3.minIndex(values)];
  const max_obs = observations[d3.maxIndex(values)];

  const [y_min, y_max] = d3.extent(data, (d) => d[name]);
  const y_domain = extents_by_measure[name];
  if (y_domain[0] > y_min) y_domain[0] = y_min;
  if (y_domain[1] < y_max) y_domain[1] = y_max;

  const x_domain = d3.extent(data, (d) => d.time);

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

    const grid_opacity = 0.1;
    const x_axis_offset = 5;
    svg
      .append("g")
      .attr(
        "transform",
        `translate(0,${chart_h - margin_bottom + x_axis_offset})`
      )
      .call(x_axis)
      .call((g) => {
        g.selectAll(".tick line")
          .attr("y2", margin_top + margin_bottom - x_axis_offset * 2 - chart_h)
          .attr("stroke-opacity", grid_opacity);
      })
      .call((g) => {
        g.selectAll(".tick text").attr("y", 5);
      })
      .call((g) => g.select(".domain").remove());

    svg
      .append("g")
      .attr("transform", `translate(${margin_left},0)`)
      .call(y_axis)
      .call((g) => g.select(".domain").remove())
      .call((g) => {
        g.selectAll(".tick line")
          .attr("x2", chart_w - margin_left - margin_right)
          .attr("stroke-opacity", grid_opacity);
      })
      .call((g) => {
        const axis_ticks = g.selectAll(".tick text").attr("x", -2);
        const num_ticks = axis_ticks.size();
        axis_ticks.each(function (d, i) {
          if (i != num_ticks - 1) return;
          d3.select(this)
            .append("tspan")
            .text(` ${unit}`)
            .style("opacity", 0.5)
            .style("font-size", "0.9em");
        });
      });

    svg
      .append("path")
      .attr("fill", "none")
      .attr("stroke", "currentColor")
      .attr("stroke-width", 1.5)
      .attr("stroke-opacity", 0.65)
      .attr("d", line(observations));

    const anotate_point = (min_or_max: "min" | "max") => {
      const obs = min_or_max == "min" ? min_obs : max_obs;
      const x_pos = x_scale(obs.time);
      const y_pos = y_scale(obs.val);
      const above = min_or_max === "max";
      const text_offset = 15 * (above ? 1 : -1);
      const pad = 2;

      svg
        .append("line")
        .attr("stroke", "currentcolor")
        .style("opacity", 0.3)
        .attr("stroke-width", 1)
        .attr("x1", x_pos)
        .attr("x2", x_pos)
        .attr("y1", y_pos + (above ? -1 : 1) * pad)
        .attr("y2", y_pos - text_offset + (above ? 1 : -1) * pad);

      const span_el = min_or_max == "max" ? max_annotation : min_annotation;
      span_el.style.left = `${x_pos + container_pad}px`;
      span_el.style.bottom = `${
        chart_h - y_pos + container_pad + text_offset
      }px`;
    };

    anotate_point("max");
    anotate_point("min");
    el.append(svg.node());
  });
</script>

<div class="container" style="--container-pad:{container_pad}px">
  <h2>
    {name} : {current_value}
    <span class="units">{unit}</span>
  </h2>

  <div class="line-chart" bind:this={el} />
  <span class="annotation" bind:this={min_annotation}
    >{format_num(min_obs.val)}</span
  >
  <span class="annotation" bind:this={max_annotation}
    >{format_num(max_obs.val)}</span
  >
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

    --plot-bg-hsl: 180 50% 93%;
    margin-inline: auto;
    margin-block: 15px;
    box-shadow: var(--shadow-elevation-medium);
    background-color: hsl(var(--plot-bg-hsl));
    color: hsl(248 31% 15%);
    width: min(500px, 100%);
    /* This variable gets set with inline css variable in the html template */
    padding: var(--container-pad, 0px);

    border-radius: 3px;
    position: relative;
  }

  h2 {
    font-weight: 200;
    text-align: right;
    margin: 0;
    margin-bottom: 5px;
    font-size: 1.6rem;
    letter-spacing: 1px;
  }

  .units {
    opacity: 0.75;
    font-size: 0.8rem;
  }

  .line-chart {
    height: 150px;
  }

  .annotation {
    font-size: 0.8rem;
    font-weight: 300;
    padding: 1px;
    background-color: hsla(var(--plot-bg-hsl) / 80%);
    position: absolute;
    transform: translate(-50%, 50%);
  }
</style>
