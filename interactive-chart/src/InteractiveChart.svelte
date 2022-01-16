<script>
	import Axis from './Axis.svelte';
	import PointInteractive from './PointInteractive.svelte';
	import {scaleTime, scaleLinear} from 'd3-scale';
	import {max, extent, bisector} from 'd3-array';
	import * as d3 from 'd3';

    export let data;
	export let margin = {top: 20, right: 5, bottom: 20, left: 5};
	export let format;
	export let key;
	export let color;
	export let title;
	export let desc;
	export let layout;

	let datum1, datum2, width, height;

	let showingCorrectChart = false;

	let userPoints = [];

	let referencePoint;

	let mouseIsPressed = 0;

	const resetCharts = () => {

		showingCorrectChart = false;

		d3.select("#realChart").selectAll("*").remove();
		d3.select("#userChart").selectAll("*").remove();

		referencePoint = createLookup(data, (d) => d.x, () => false);

		userPoints.splice(0, userPoints.length)

		userPoints.push({
			x: data[0].x,
			y: data[0].y
		}); //Take the first point

		referencePoint[data[0].x] = true;
	}

	const drawChart = (data, selector, color, final) => {
		let chart =  d3.select(selector);

		let path = d3.line()
			.x(d => x(d[key.x]))
			.y(d => y(d[key.y]))
			.curve(d3.curveMonotoneX);

		chart.selectAll("*").remove();

		if(!final){
			return chart.append("path")
			.attr("d", path(data))
			.attr("stroke", color)
			.style("stroke-dasharray", ("3, 3"))
			.attr("stroke-width", "4")
			.attr("fill", "none");
		}else
		{
			return chart.append("path")
			.attr("d", path(data))
			.attr("stroke", color)
			.attr("stroke-width", "4")
			.attr("fill", "none");
		}
	}

	let  createLookup = (arr, keyFn, valFn) => {

		const lookup = {};

		arr.forEach(d => {
			const key = keyFn(d);
			lookup[key] = valFn(d);
		});

		return lookup;
  	}

	let findLoc = (el, arr, st, en) => {
            st = st || 0;
            en = en || arr.length;
            for (let i = 0; i < arr.length; i++) {
                if (x(arr[i].x) > x(el))
                    return i - 1;
            }
            return en;
    }

	let findClosestPoint = (pointsSet, m) => {
		const mX = (m.offsetX) ? m.offsetX : m.clientX;
		const _data = [...pointsSet];
		_data.sort((a,b) => a[key.x] - b[[key.x]]);
		const index = x.invert(mX);
		const i = bisector(d => d[key.x]).center(_data, index);

		return _data[i];
	}

	let showRealChart = () => {

		var path = drawChart(data, "#realChart", color[1], true);

		var totalLength = path.node().getTotalLength();

		//Animation
		path
        .attr("stroke-dasharray", totalLength + " " + totalLength)
        .attr("stroke-dashoffset", totalLength)
        .transition()
          .duration(4000)
          .ease(d3.easeLinear)
          .attr("stroke-dashoffset", 0)

		showingCorrectChart = true;

	}

	const handleClick = (m) => {

		let cursorPosition = d3.pointer(m); //Select the cursor position

		//Create a new point
		//Todo -- Insert to the nearest reference position
		let closestReferencePoint = findClosestPoint(data, m)

		let outsideLimit = cursorPosition[1] >= height - margin.top - margin.bottom;
		if(!referencePoint[closestReferencePoint.x] && !outsideLimit){

			//Find where to insert the element sorted
			userPoints.splice(findLoc(closestReferencePoint.x, userPoints) + 1, 0, {x: closestReferencePoint.x, y: y.invert(cursorPosition[1])});

			//Check the point as used
			referencePoint[closestReferencePoint.x] = true;
		}
		else if(!outsideLimit){

			//Update the nearest point y position
			datum1.y = y.invert(cursorPosition[1]); //Datum is calculated on mouse move, so that is a reference

		}

		drawChart(userPoints, "#userChart", color[0], false)


	}

	const mouseMove = (m) => {

		//The real chart is showing?
		if(showingCorrectChart){
			datum2 = findClosestPoint(data, m) //Find the datum
		}

		//find the nearest point to the cursor
		datum1 = findClosestPoint(userPoints, m)

		//If the mouse is pressed handleClick
		if(mouseIsPressed === 1) handleClick(m);

	}

	const leave = (m) => {
		datum1 = undefined;
		datum2 = undefined;
		mouseIsPressed = 0;
	}

	//Generate the initial data
	data = data.map(d => ({x: new Date(d.x), y: d.y}));

	data.sort((a,b) => b.x-a.x).reverse();

	referencePoint = createLookup(data, (d) => d.x, () => false);

	//Set the initial point
	userPoints.push({
		x: data[0].x,
		y: data[0].y
	});
	referencePoint[data[0].x] = true;


	$:  x = scaleTime()
			.domain(extent(data, d => d[key.x]))
			.range([margin.left, width - margin.right]);

	$: y = scaleLinear()
			.domain([0, max(data, d => d[key.y])])
			.range([height - margin.bottom - margin.top, margin.top]);

	$:
		if (width || height) {
			drawChart(userPoints, "#userChart", color[0], false)
			drawChart(data, "#realChart", color[1], true)
		}


</script>


<div class='graphic {layout} center' bind:clientWidth={width} bind:clientHeight={height}>
{#if width}

<svg xmlns:svg='https://www.w3.org/2000/svg'
	viewBox='0 0 {width} {height}'
	{width}
	{height}
	aria-labelledby='title desc'
	on:click={handleClick}
	on:touchmove|preventDefault
	on:pointermove|preventDefault={mouseMove}
	on:mouseleave={leave}
	on:touchend={leave}
	on:mousedown={() => {mouseIsPressed = 1}}
	on:mouseup={() => mouseIsPressed = 0}
	>
	<title id='title'>{title}</title>
	<desc id='desc'>{desc}</desc>

	<Axis {width} {height} {margin} scale={y} position='left' format={format.y} />
	<Axis {width} {height} {margin} scale={x} position='bottom' format={format.x} />




	<g id='userChart'/>
	<g id='realChart'/>


	<PointInteractive datum = {datum1} {format} {x} {y} {key} {width} />
	<PointInteractive datum ={datum2} {format} {x} {y} {key} {width} />




</svg>

<div class="button" on:click={showRealChart}>
	Show real path
</div>

<div class="button" on:click={resetCharts}>
	Clear chart
</div>

{/if}
</div>

<style>
	svg{
		cursor: url('/img/pen.png') 4 36, pointer;
		width: 100%;
		height: 100%;
		background-color: rgb(241, 241, 241);
	}

	:global(.graphic) {
		height:100%;
		width: 100%;
		margin-bottom:3rem;
	}

	.button{
		text-align: center;
		display: inline-block;
		outline: none;
		cursor: pointer;
		font-weight: 500;
		border: 1px solid transparent;
		border-radius: 2px;
		height: 36px;
		line-height: 34px;
		font-size: 14px;
		color: #ffffff;
		background-color: #c4c4c4;
		transition: background-color 0.2s ease-in-out 0s, opacity 0.2s ease-in-out 0s;
		padding: 0 18px;
		font-family: Arial, Helvetica, sans-serif;
	}
	.button:hover {
		color: #ffffff;
		background-color: #8d8d8d;
	}

	.center{
		text-align: center;
	}
</style>