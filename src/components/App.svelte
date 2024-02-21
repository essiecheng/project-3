<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import Papa from 'papaparse';

  let data;

  async function convertCsvToJson() {
    const res = await fetch('olympic.csv');
    const csv = await res.text();

    const parsed = Papa.parse(csv, { header: true });
    const data = { name: 'olympic', children: [] };

    parsed.data.forEach((entry) => {
      let hasNaN = false;
      for (let key in entry) {
        if ((entry[key] === 'NaN') | (entry[key] === '')) {
          hasNaN = true;
          break;
        }
      }
      if (hasNaN) {
        return;
      }
      const year = entry.Year;
      const sport = entry.Sport;
      const country = entry.Country;
      const athlete = entry.Athlete;
      const medal = entry.Medal;

      let sportNode = data.children.find((y) => y.name === sport);
      if (!sportNode) {
        sportNode = { name: sport, children: [] };
        data.children.push(sportNode);
      }

      let yearNode = sportNode.children.find((s) => s.name === year);
      if (!yearNode) {
        yearNode = { name: year, children: [] };
        sportNode.children.push(yearNode);
      }

      let countryNode = yearNode.children.find((c) => c.name === country);
      if (!countryNode) {
        countryNode = { name: country, children: [] };
        yearNode.children.push(countryNode);
      }

      let athleteNode = countryNode.children.find((c) => c.name === athlete);
      if (!athleteNode) {
        athleteNode = { name: athlete, children: [] };
        countryNode.children.push(athleteNode);
      }

      // Group by medal type
      let medalType;
      if (medal === 'Gold') {
        medalType = 'Gold';
      } else if (medal === 'Silver') {
        medalType = 'Silver';
      } else if (medal === 'Bronze') {
        medalType = 'Bronze';
      }

      // Find or create the medal type node
      let medalNode = athleteNode.children.find((m) => m.name === medalType);
      if (!medalNode) {
        medalNode = { name: medalType, children: [] };
        athleteNode.children.push(medalNode);
      }

      // Increase the count of the medal type
      medalNode.value = (medalNode.value || 0) + 1;
    });

    return data;
  }

  let svg;
  let y; // Define y scale

  onMount(async () => {
    data = await convertCsvToJson(); // Await the JSON data
    console.log(data);
    createChart(data);
  });

  function createChart() {
    const width = 1500;
    const height = 800;
    const marginTop = 30;
    const marginRight = 30;
    const marginBottom = 0;
    const marginLeft = 100;
    const barStep = 27;

    const root = _root(d3, data);
    const color = _color(d3);

    const x = _x(d3, marginLeft, width, marginRight);
    const y = d3.scaleBand().range([0, height]).padding(_barPadding(barStep));

    const xAxis = _xAxis(marginTop, d3, x, width);
    const yAxis = _yAxis(marginLeft, marginTop, height, marginBottom);

    const stack = _stack(x, barStep);
    const stagger = _stagger(x, barStep);

    const duration = _duration();

    svg = _chart(
      d3,
      width,
      height,
      x,
      root,
      _up(
        duration,
        x,
        d3,
        xAxis,
        stagger,
        stack,
        color,
        _bar(marginTop, barStep, _barPadding(barStep), marginLeft, x),
        _down(
          d3,
          duration,
          _bar(marginTop, barStep, _barPadding(barStep), marginLeft, x),
          stack,
          stagger,
          x,
          xAxis,
          barStep,
          color
        ),
        barStep
      ),
      xAxis,
      yAxis,
      _down(
        d3,
        duration,
        _bar(marginTop, barStep, _barPadding(barStep), marginLeft, x),
        stack,
        stagger,
        x,
        xAxis,
        barStep,
        color
      )
    );

    document.querySelector('#chart-container').appendChild(svg);
  }

  function _chart(d3, width, height, x, root, up, xAxis, yAxis, down) {
    const svg = d3
      .create('svg')
      .attr('viewBox', [0, 0, width, height])
      .attr('width', width)
      .attr('height', height)
      .attr('style', 'max-width: 100%; height: auto;');

    x.domain([0, root.value]);

    svg
      .append('rect')
      .attr('class', 'background')
      .attr('fill', 'none')
      .attr('pointer-events', 'all')
      .attr('width', width)
      .attr('height', height)
      .attr('cursor', 'pointer')
      .on('click', (event, d) => up(svg, d));

    svg.append('g').call(xAxis);

    svg.append('g').call(yAxis);

    down(svg, root);

    return svg.node();
  }

  function _down(d3, duration, bar, stack, stagger, x, xAxis, barStep, color) {
    return function down(svg, d) {
      if (!d.children || d3.active(svg.node())) return;

      svg.select('.background').datum(d);

      const transition1 = svg.transition().duration(duration);
      const transition2 = transition1.transition();

      const exit = svg.selectAll('.enter').attr('class', 'exit');

      exit.selectAll('rect').attr('fill-opacity', (p) => (p === d ? 0 : null));

      exit.transition(transition1).attr('fill-opacity', 0).remove();

      const enter = bar(svg, down, d, '.y-axis').attr('fill-opacity', 0);

      enter.transition(transition1).attr('fill-opacity', 1);

      enter
        .selectAll('g')
        .attr('transform', stack(d.index))
        .transition(transition1)
        .attr('transform', stagger());

      x.domain([0, d3.max(d.children, (d) => d.value)]);

      svg.selectAll('.x-axis').transition(transition2).call(xAxis);

      enter
        .selectAll('g')
        .transition(transition2)
        .attr('transform', (d, i) => `translate(0,${barStep * i})`);

      enter
        .selectAll('rect')
        .attr('fill', color(true))
        .attr('fill-opacity', 1)
        .transition(transition2)
        .attr('fill', (d) => color(!!d.children))
        .attr('width', (d) => x(d.value) - x(0));
    };
  }

  function _up(
    duration,
    x,
    d3,
    xAxis,
    stagger,
    stack,
    color,
    bar,
    down,
    barStep
  ) {
    return function up(svg, d) {
      if (!d.parent || !svg.selectAll('.exit').empty()) return;

      svg.select('.background').datum(d.parent);

      const transition1 = svg.transition().duration(duration);
      const transition2 = transition1.transition();

      const exit = svg.selectAll('.enter').attr('class', 'exit');

      x.domain([0, d3.max(d.parent.children, (d) => d.value)]);

      svg.selectAll('.x-axis').transition(transition1).call(xAxis);

      exit.selectAll('g').transition(transition1).attr('transform', stagger());

      exit
        .selectAll('g')
        .transition(transition2)
        .attr('transform', stack(d.index));

      exit
        .selectAll('rect')
        .transition(transition1)
        .attr('width', (d) => x(d.value) - x(0))
        .attr('fill', color(true));

      exit.transition(transition2).attr('fill-opacity', 0).remove();

      const enter = bar(svg, down, d.parent, '.exit').attr('fill-opacity', 0);

      enter
        .selectAll('g')
        .attr('transform', (d, i) => `translate(0,${barStep * i})`);

      enter.transition(transition2).attr('fill-opacity', 1);

      enter
        .selectAll('rect')
        .attr('fill', (d) => color(!!d.children))
        .attr('fill-opacity', (p) => (p === d ? 0 : null))
        .transition(transition2)
        .attr('width', (d) => x(d.value) - x(0))
        .on('end', function (p) {
          d3.select(this).attr('fill-opacity', 1);
        });
    };
  }

  function _stack(x, barStep) {
    return function stack(i) {
      let value = 0;
      return (d) => {
        const t = `translate(${x(value) - x(0)},${barStep * i})`;
        value += d.value;
        return t;
      };
    };
  }

  function _root(d3, data) {
    return d3
      .hierarchy(data)
      .sum((d) => d.value)
      .sort((a, b) => b.value - a.value)
      .eachAfter(
        (d) =>
          (d.index = d.parent ? (d.parent.index = d.parent.index + 1 || 0) : 0)
      );
  }

  function _x(d3, marginLeft, width, marginRight) {
    return d3.scaleLinear().range([marginLeft, width - marginRight]);
  }

  function _xAxis(marginTop, d3, x, width) {
    return (g) =>
      g
        .attr('class', 'x-axis')
        .attr('transform', `translate(0,${marginTop})`)
        .call(d3.axisTop(x).ticks(width / 80, 's'))
        .call((g) =>
          (g.selection ? g.selection() : g).select('.domain').remove()
        );
  }

  function _yAxis(marginLeft, marginTop, height, marginBottom) {
    return (g) =>
      g
        .attr('class', 'y-axis')
        .attr('transform', `translate(${marginLeft + 0.5},0)`)
        .call((g) =>
          g
            .append('line')
            .attr('stroke', 'currentColor')
            .attr('y1', marginTop)
            .attr('y2', height - marginBottom)
        );
  }

  function _color(d3) {
    return d3.scaleOrdinal([true, false], ['steelblue', '#aaa']);
  }

  function _bar(marginTop, barStep, barPadding, marginLeft, x) {
    return function bar(svg, down, d, selector) {
      const g = svg
        .insert('g', selector)
        .attr('class', 'enter')
        .attr('transform', `translate(0,${marginTop + barStep * barPadding})`)
        .attr('text-anchor', 'end')
        .style('font', '10px sans-serif');

      const bar = g
        .selectAll('g')
        .data(d.children)
        .join('g')
        .attr('cursor', (d) => (!d.children ? null : 'pointer'))
        .on('click', (event, d) => down(svg, d));

      bar
        .append('text')
        .attr('x', marginLeft - 6)
        .attr('y', (barStep * (1 - barPadding)) / 2)
        .attr('dy', '.35em')
        .text((d) => d.data.name);

      bar
        .append('rect')
        .attr('x', x(0))
        .attr('width', (d) => x(d.value) - x(0))
        .attr('height', barStep * (1 - barPadding));

      return g;
    };
  }

  function _stagger(x, barStep) {
    return function stagger() {
      let value = 0;
      return (d, i) => {
        const t = `translate(${x(value) - x(0)},${barStep * i})`;
        value += d.value;
        return t;
      };
    };
  }

  function _barPadding(barStep) {
    return 3 / barStep;
  }

  function _duration() {
    return 750;
  }
</script>

<main>
  <h1>Hierarchical bar chart</h1>
  <script
    src="https://cdnjs.cloudflare.com/ajax/libs/PapaParse/5.3.0/papaparse.min.js"
  ></script>
  <div id="chart-container"></div>
</main>
