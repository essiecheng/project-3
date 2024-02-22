<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import Papa from 'papaparse';

  let data;
  let gold = true;
  let silver = true;
  let bronze = true;

  async function convertCsvToJson() {
    const res = await fetch('olympic.csv');
    const csv = await res.text();

    const parsed = Papa.parse(csv, { header: true });
    const data = { name: 'olympic', children: [] };

    // Clean data
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
      const gender = entry.Gender;
      const medal = entry.Medal;
      const event = entry.Event;

      // Define hierarchy
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

      let genderNode = countryNode.children.find((c) => c.name === gender);
      if (!genderNode) {
        genderNode = { name: gender, children: [] };
        countryNode.children.push(genderNode);
      }

      let athleteNode = genderNode.children.find((c) => c.name === athlete);
      if (!athleteNode) {
        athleteNode = { name: athlete, children: [] };
        genderNode.children.push(athleteNode);
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

      // Continue defining hierarchy
      let medalNode = athleteNode.children.find((m) => m.name === medalType);
      if (!medalNode) {
        medalNode = { name: medalType, children: [] };
        athleteNode.children.push(medalNode);
      }

      let eventNode = medalNode.children.find((c) => c.name === event);
      if (!eventNode) {
        eventNode = { name: event, children: [] };
        medalNode.children.push(eventNode);
      }

      eventNode.value = (eventNode.value || 0) + 1;
    });
    return data;
  }

  // Query filter function for Medal
  function queryData() {
    let filteredData = JSON.parse(JSON.stringify(data));

    if (!gold) {
      filteredData = filterMedal(filteredData, 'Gold');
    }
    if (!silver) {
      filteredData = filterMedal(filteredData, 'Silver');
    }
    if (!bronze) {
      filteredData = filterMedal(filteredData, 'Bronze');
    }
    return filteredData;
  }

  // Helper function to filter out specific medal type
  function filterMedal(data, medalType) {
    return {
      ...data,
      children: data.children.map((sportNode) => ({
        ...sportNode,
        children: sportNode.children.map((yearNode) => ({
          ...yearNode,
          children: yearNode.children.map((countryNode) => ({
            ...countryNode,
            children: countryNode.children.map((genderNode) => ({
              ...genderNode,
              children: genderNode.children.map((athleteNode) => ({
                ...athleteNode,
                children: athleteNode.children.filter(
                  (medalNode) => medalNode.name !== medalType
                ),
              })),
            })),
          })),
        })),
      })),
    };
  }

  // Update chart with queried data
  function updateChart() {
    const filteredData = queryData();
    svg.remove();
    createChart(filteredData);
  }

  let svg;

  onMount(async () => {
    data = await convertCsvToJson();
    createChart(data);
  });

  // Create chart
  function createChart(data) {
    const width = 1500;
    const height = 900;
    const marginTop = 50;
    const marginRight = 40;
    const marginBottom = 0;
    const marginLeft = 190;
    const barStep = 27;

    const root = _root(d3, data);
    const color = _color(d3);

    const x = _x(d3, marginLeft, width, marginRight);
    const xAxis = _xAxis(marginTop, d3, x, width);

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

  // Helper function to label y-axis for each layer
  function getYAxisTitleText(depth) {
    if (depth == 1) {
      return 'Sport';
    } else if (depth == 2) {
      return 'Year';
    } else if (depth == 3) {
      return 'Country';
    } else if (depth == 4) {
      return 'Gender';
    } else if (depth == 5) {
      return 'Athlete';
    } else if (depth == 6) {
      return 'Medal';
    } else {
      return 'Event';
    }
  }

  // Chart
  function _chart(d3, width, height, x, root, up, xAxis, down) {
    const svg = d3
      .create('svg')
      .attr('viewBox', [0, 0, width, height])
      .attr('width', width)
      .attr('height', height)
      .attr('style', 'max-width: 100%; height: auto;');

    x.domain([0, root.value]);

    // Move up the layer when we click the background
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

    down(svg, root);

    // Label x-axis
    svg
      .append('text')
      .attr('class', 'x-axis-title')
      .attr('text-anchor', 'middle')
      .attr('x', width / 2)
      .attr('y', height / 50)
      .text('Medal Count')
      .style('font-size', '20px')
      .style('font-family', 'serif');

    return svg.node();
  }

  // Down
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

  // Up
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

  // Stack
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

  // Root
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
        )
        .style('font-family', 'serif')
        .style('font-size', '13px');
  }

  // Set the color of the bar
  function _color(d3) {
    return d3.scaleOrdinal([true, false], ['#66BD48', '#66BD48']);
  }

  // Highlight when hovered
  let hoveredBar = null;
  function handleBarHover(bar) {
    hoveredBar = bar;
    d3.select(bar).select('rect').attr('fill', 'orange');
  }

  // Unhighlight when mouse leaves
  function handleBarMouseLeave(bar) {
    hoveredBar = null;
    d3.select(bar).select('rect').attr('fill', '#66BD48');
  }

  // Add features to the bars
  function _bar(marginTop, barStep, barPadding, marginLeft, x) {
    return function bar(svg, down, d, selector) {
      // track the depth of each layer
      let depth = 1;
      let currentNode = d;
      while (currentNode.parent) {
        depth++;
        currentNode = currentNode.parent;
      }

      const g = svg
        .insert('g', selector)
        .attr('class', 'enter')
        .attr('transform', `translate(0,${marginTop + barStep * barPadding})`)
        .attr('text-anchor', 'end')
        .style('font', '10px sans-serif');

      const bars = g
        .selectAll('.bar-group')
        .data(d.children)
        .join('g')
        .attr('class', 'bar-group')
        .attr('cursor', (d) => (!d.children ? null : 'pointer'))
        .on('click', (event, d) => down(svg, d))
        .on('mouseover', function (event, d) {
          handleBarHover(this);

          // Show tooltip on mouseover
          const tooltip = document.querySelector('.tooltip');
          let tooltipContent = '';
          if (d.depth === 1) {
            tooltipContent = `
                Sport: ${d.data.name}<br>
                Medal Count: ${d.value}`;
          } else if (d.depth === 2) {
            tooltipContent = `
                Sport: ${d.parent.data.name}<br>
                Year: ${d.data.name}<br>
                Medal Count: ${d.value}`;
          } else if (d.depth === 3) {
            tooltipContent = `
                Sport: ${d.parent.parent.data.name}<br>
                Year: ${d.parent.data.name}<br>
                Country: ${d.data.name}<br>
                Medal Count: ${d.value}`;
          } else if (d.depth === 4) {
            tooltipContent = `
                Sport: ${d.parent.parent.parent.data.name}<br>
                Year: ${d.parent.parent.data.name}<br>
                Country: ${d.parent.data.name}<br>
                Gender: ${d.data.name}<br>
                Medal Count: ${d.value}`;
          } else if (d.depth === 5) {
            tooltipContent = `
                Sport: ${d.parent.parent.parent.parent.data.name}<br>
                Year: ${d.parent.parent.parent.data.name}<br>
                Country: ${d.parent.parent.data.name}<br>
                Gender: ${d.parent.data.name}<br>
                Athlete: ${d.data.name}<br>
                Medal Count: ${d.value}`;
          } else if (d.depth === 6) {
            tooltipContent = `
                Sport: ${d.parent.parent.parent.parent.parent.data.name}<br>
                Year: ${d.parent.parent.parent.parent.data.name}<br>
                Country: ${d.parent.parent.parent.data.name}<br>
                Gender: ${d.parent.parent.data.name}<br>
                Athlete: ${d.parent.data.name}<br>
                Medal: ${d.data.name}<br>
                Medal Count: ${d.value}`;
          } else {
            tooltipContent = `
                Sport: ${d.parent.parent.parent.parent.parent.parent.data.name}<br>
                Year: ${d.parent.parent.parent.parent.parent.data.name}<br>
                Country: ${d.parent.parent.parent.parent.data.name}<br>
                Gender: ${d.parent.parent.parent.data.name}<br>
                Athlete: ${d.parent.parent.data.name}<br>
                Medal: ${d.parent.data.name}<br>
                Event: ${d.data.name}`;
          }

          tooltip.innerHTML = tooltipContent;
          tooltip.style.opacity = 1;
        })
        .on('mousemove', function (event) {
          // Position tooltip
          const tooltip = document.querySelector('.tooltip');
          const chartContainer = document.querySelector('#chart-container');

          const containerRect = chartContainer.getBoundingClientRect();
          const offsetX = event.pageX - containerRect.left;
          const offsetY = event.pageY - containerRect.top - window.scrollY;

          tooltip.style.left = offsetX + 'px';
          tooltip.style.top = offsetY + 'px';
        })
        .on('mouseleave', function () {
          handleBarMouseLeave(this);
          // Hide tooltip on when the mouse leaves
          const tooltip = document.querySelector('.tooltip');
          tooltip.style.opacity = 0;
        });

      // Label each bar
      bars
        .append('text')
        .attr('class', 'bar-label')
        .attr('x', marginLeft - 6)
        .attr('y', (barStep * (1 - barPadding)) / 2)
        .attr('dy', '.35em')
        .attr('font-family', 'serif')
        .style('font-size', '14px')
        .text((d) => d.data.name);

      bars
        .append('rect')
        .attr('x', marginLeft)
        .attr('width', (d) => x(d.value))
        .attr('height', barStep * (1 - barPadding))
        .attr('fill', (d) => (d === hoveredBar ? 'red' : '#66BD48'));

      // Update y-axis for each layer
      const yAxisText = g
        .append('text')
        .attr('class', 'bar-axis')
        .attr('x', marginLeft - 530)
        .attr('y', 30)
        .attr('dy', '.35em')
        .attr('font-size', '20px')
        .attr('font-family', 'serif')
        .style('text-anchor', 'middle')
        .attr('transform', 'rotate(-90)')
        .text(() => getYAxisTitleText(depth));

      return g;
    };
  }

  // Stagger
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

  // Bar padding
  function _barPadding(barStep) {
    return 3 / barStep;
  }

  // Duration
  function _duration() {
    return 750;
  }
</script>

<main>
  <div class="banner">
    <h1>Medals and Memories: Charting Olympic Greatness</h1>
    <h3>
      This chart contains information about medals won for each sport, year,
      country, gender, and event between the period of 1976-2008 in the Summer
      Olympics.
    </h3>
    <div class="link">
      <p>Click a bar to drill down and click the background to go back up.</p>
      <a
        href="https://docs.google.com/document/d/1yUMUHjXo-1xp1h4csNjPcgE3yqoF7W3_jrWIG0RTXgM/edit?usp=sharing"
      >
        Project Write-Up</a
      >
    </div>
  </div>
  <script
    src="https://cdnjs.cloudflare.com/ajax/libs/PapaParse/5.3.0/papaparse.min.js"
  ></script>
  <div id="chart-container" style="width: 100%; height: 100;">
    <div class="tooltip" style="opacity: 0;"></div>
    <div>
      <label class="gold-label">
        <input type="checkbox" bind:checked={gold} on:change={updateChart} /> Gold
        🥇
      </label>
      <label class="silver-label">
        <input type="checkbox" bind:checked={silver} on:change={updateChart} /> Silver
        🥈
      </label>
      <label class="bronze-label">
        <input type="checkbox" bind:checked={bronze} on:change={updateChart} /> Bronze
        🥉
      </label>
    </div>
  </div>
</main>

<style>
  .tooltip {
    position: absolute;
    background-color: white;
    border: 1px solid #aaa;
    padding: 10px;
    pointer-events: none;
    z-index: 999;
  }

  #chart-container {
    background-image: url('https://1000logos.net/wp-content/uploads/2021/03/Olympics-logo.png');
    background-size: 20%;
    background-repeat: no-repeat;
    background-position: 95% 90%;
    width: 100%;
    max-width: 1200px;
    height: 100%;
    position: relative;
    border: 2px solid transparent;
    box-sizing: border-box;
    margin: 0 auto;
  }

  #chart-container::before {
    content: '';
    position: absolute;
    top: -4px;
    left: -2px;
    right: -2px;
    bottom: -2px;
    border: 5px solid;
    border-image: linear-gradient(
      to bottom right,
      #0081c8,
      #fcb131,
      #00a651,
      #ee334e
    );
    border-image-slice: 1;
    z-index: -1;
  }

  h1 {
    font-size: 43px;
    font-weight: 500, 'bold';
    text-transform: uppercase;
    line-height: 1;
    text-align: center;
    font-family: 'verdana';
  }

  h3 {
    font-family: 'verdana';
    text-align: center;
    font-size: 20px;
    line-height: 28px;
  }

  p {
    font-family: 'verdana';
    letter-spacing: 0.55px;
    font-size: 16px;
    line-height: 28px;
    text-align: center;
  }

  .link {
    text-align: center;
    font-family: 'verdana';
  }

  .banner {
    background-image: linear-gradient(
      to bottom right,
      rgba(0, 129, 200, 0.5),
      rgba(252, 177, 49, 0.5),
      rgba(0, 166, 81, 0.5),
      rgba(238, 51, 78, 0.5)
    );
    padding: 20px;
    margin-bottom: 20px;
    border-radius: 10px;
    color: black;
  }
  .gold-label {
    background-color: gold;
    padding: 5px 5px;
    border-radius: 5px;
  }
  .silver-label {
    background-color: silver;
    padding: 5px 5px;
    border-radius: 5px;
  }
  .bronze-label {
    background-color: #cd7f32;
    padding: 5px 5px;
    border-radius: 5px;
  }
  #chart-container div {
    margin-top: 20px;
    margin-left: 10px;
  }
</style>
