<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let unemployment = [];

  onMount(async () => {
    const res = await fetch('bls-metro-unemployment.csv');
    const csv = await res.text();
    unemployment = d3.csvParse(csv, d3.autoType);
    console.log(unemployment);
    createChart(); // Call the createChart function after data is fetched
  });

  function createChart() {
    const width = 928;
    const height = 600;
    const marginTop = 20;
    const marginRight = 20; // Corrected variable name
    const marginBottom = 30;
    const marginLeft = 30;

    const x = d3
      .scaleUtc()
      .domain(d3.extent(unemployment, (d) => d.date))
      .range([marginLeft, width - marginRight]); // Corrected variable name

    const y = d3
      .scaleLinear()
      .domain([0, d3.max(unemployment, (d) => d.unemployment)])
      .nice()
      .range([height - marginBottom, marginTop]);

    const svg = d3
      .select('#chart-container')
      .append('svg')
      .attr('width', width)
      .attr('height', height)
      .attr('viewBox', [0, 0, width, height])
      .attr(
        'style',
        'max-width: 100%; height: auto; overflow: visible; font: 10px sans-serif;'
      );

    svg
      .append('g')
      .attr('transform', `translate(0,${height - marginBottom})`)
      .call(
        d3
          .axisBottom(x)
          .ticks(width / 80)
          .tickSizeOuter(0)
      );

    svg
      .append('g')
      .attr('transform', `translate(${marginLeft},0)`)
      .call(d3.axisLeft(y))
      .call((g) => g.select('.domain').remove())
      .call(
        unemployment
          ? () => {}
          : (g) =>
              g
                .selectAll('.tick line')
                .clone()
                .attr('x2', width - marginLeft - marginRight)
                .attr('stroke-opacity', 0.1)
      )
      .call((g) =>
        g
          .append('text')
          .attr('x', -marginLeft)
          .attr('y', 10)
          .attr('fill', 'currentColor')
          .attr('text-anchor', 'start')
          .text('↑ Unemployment (%)')
      );

    const points = unemployment.map((d) => [
      x(d.date),
      y(d.unemployment),
      d.division,
    ]);

    if (unemployment)
      svg
        .append('path')
        .attr('fill', 'none')
        .attr('stroke', '#ccc')
        .attr(
          'd',
          d3.Delaunay.from(points).voronoi([0, 0, width, height]).render()
        );

    const groups = d3.rollup(
      points,
      (v) => Object.assign(v, { z: v[0][2] }),
      (d) => d[2]
    );

    const line = d3.line();
    const path = svg
      .append('g')
      .attr('fill', 'none')
      .attr('stroke', 'steelblue')
      .attr('stroke-width', 1.5)
      .attr('stroke-linejoin', 'round')
      .attr('stroke-linecap', 'round')
      .selectAll('path')
      .data(groups.values())
      .join('path')
      .style('mix-blend-mode', 'multiply')
      .attr('d', line);

    const dot = svg.append('g').attr('display', 'none');

    dot.append('circle').attr('r', 2.5);

    dot.append('text').attr('text-anchor', 'middle').attr('y', -8);

    svg
      .on('pointerenter', pointerentered)
      .on('pointermove', pointermoved)
      .on('pointerleave', pointerleft)
      .on('touchstart', (event) => event.preventDefault());

    return svg.node();

    function pointermoved(event) {
      const [xm, ym] = d3.pointer(event, svg.node());
      const i = d3.leastIndex(points, ([x, y]) => Math.hypot(x - xm, y - ym));
      const [x, y, k] = points[i];
      path
        .style('stroke', ({ z }) => (z === k ? null : '#ddd'))
        .filter(({ z }) => z === k)
        .raise();
      dot.attr('transform', `translate(${x},${y})`);
      dot.select('text').text(k);
      svg
        .property('value', unemployment[i])
        .dispatch('input', { bubbles: true });
    }

    function pointerentered() {
      path.style('mix-blend-mode', null).style('stroke', '#ddd');
      dot.attr('display', null);
    }

    function pointerleft() {
      path.style('mix-blend-mode', 'multiply').style('stroke', null);
      dot.attr('display', 'none');
      svg.node().value = null;
      svg.dispatch('input', { bubbles: true });
    }
  }
</script>

<main>
  <h1>hi</h1>
  <div id="chart-container"></div>
</main>
