function _bar(marginTop, barStep, barPadding, marginLeft, x) {
  return function bar(svg, down, d, selector) {
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
            Sport: ${d.data.name}`;
        } else if (d.depth === 2) {
          tooltipContent = `
            Sport: ${d.parent.data.name}<br>
            Year: ${d.data.name}`;
        } else if (d.depth === 3) {
          tooltipContent = `
            Sport: ${d.parent.parent.data.name}<br>
            Year: ${d.parent.data.name}<br>
            Country: ${d.data.name}`;
        } else if (d.depth === 4) {
          tooltipContent = `
            Sport: ${d.parent.parent.parent.data.name}<br>
            Year: ${d.parent.parent.data.name}<br>
            Country: ${d.parent.data.name}<br>
            Gender: ${d.data.name}`;
        } else if (d.depth === 5) {
          tooltipContent = `
            Sport: ${d.parent.parent.parent.parent.data.name}<br>
            Year: ${d.parent.parent.parent.data.name}<br>
            Country: ${d.parent.parent.data.name}<br>
            Gender: ${d.parent.data.name}<br>
            Athlete: ${d.data.name}`;
        } else if (d.depth === 6) {
          tooltipContent = `
            Sport: ${d.parent.parent.parent.parent.parent.data.name}<br>
            Year: ${d.parent.parent.parent.parent.data.name}<br>
            Country: ${d.parent.parent.parent.data.name}<br>
            Gender: ${d.parent.parent.data.name}<br>
            Athlete: ${d.parent.data.name}<br>
            Medal: ${d.data.name}`;
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
        // Position tooltip relative to the mouse cursor
        const tooltip = document.querySelector('.tooltip');
        tooltip.style.left = event.pageX + 'px';
        tooltip.style.top = event.pageY - window.scrollY + 'px';
      })
      .on('mouseleave', function () {
        handleBarMouseLeave(this);
        // Hide tooltip on mouseleave
        const tooltip = document.querySelector('.tooltip');
        tooltip.style.opacity = 0;
      });

    bars
      .append('text')
      .attr('class', 'bar-label')
      .attr('x', marginLeft - 6)
      .attr('y', (barStep * (1 - barPadding)) / 2)
      .attr('dy', '.35em')
      .text((d) => d.data.name);

    bars
      .append('text') // Add text element for displaying value
      .attr('class', 'bar-value')
      .attr('x', (d) => x(d.value) + 5) // Adjust position for value text
      .attr('y', (barStep * (1 - barPadding)) / 2)
      .attr('dy', '.35em')
      .text((d) => d.value); // Display value

    bars
      .append('rect')
      .attr('x', marginLeft)
      .attr('width', (d) => x(d.value))
      .attr('height', barStep * (1 - barPadding))
      .attr('fill', (d) => (d === hoveredBar ? 'red' : '#66BD48'));

    return g;
  };
}
