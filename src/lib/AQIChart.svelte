<script lang="ts">
	import * as d3 from 'd3';

	interface Item {
		city: string;
		country: string;
		mainPollutant: string;
		pm25: number;
		state: string;
		stationName: string;
		timestamp: Date;
		usAqi: number;
	}

	// properties this component accepts
	let { data, thisStation= $bindable() }: { data: Item[], thisStation: string | null } = $props();

	$effect(() => {
		console.log('AQIChart thisStation:', thisStation);
	});

	// chart dimension variables
	let width = $state(700);
	let height = $state(400);
	let margin = $state({top: 40, right: 20, bottom: 80, left: 40});

	let usableArea = $derived({
		top: margin.top,
		right: width - margin.right,
		bottom: height - margin.bottom,
		left: margin.left
	});

	//Colored AQI definitions
	const aqiLevels = [
		{ name: 'Good', min: 0, max: 50, color: '#9cd84e' },
		{ name: 'Moderate', min: 50, max: 100, color: '#facf39' },
		{ name: 'Unhealthy for Sensitive Groups', min: 100, max: 150, color: '#f99049' },
		{ name: 'Unhealthy', min: 150, max: 200, color: '#f65e5f' },
		{ name: 'Very Unhealthy', min: 200, max: 300, color: '#a070b6' },
		{ name: 'Hazardous', min: 300, color: '#a06a7b' }
	];



	//checkbox true/false for showing raw data
	let dataPointsOn = $state(false);

	//filter to all of current station's data (for raw data display)
	//if a current station is selected, filter selected data to only the current station's data
	//otherwise, return all of the data sorted by date (fixes gaps in shading & line)
	const selectedData = $derived(
		thisStation 
			? data.filter((d) => d.stationName === thisStation)
			: data.sort((a, b) => d3.ascending(a.timestamp, b.timestamp))
	);

	//function for grouping data by months
	function findMonthData (data: Item[]) {
		return Array.from(
			d3.rollup(
				data,
				//calculates the mean of AQIs for each month
				(v) => ({
					mean: d3.mean(v, (d) => d.usAqi),
					firstQuantile: d3.quantile(v.map((d) => d.usAqi).sort(d3.ascending), 0.1),
					lastQuantile: d3.quantile(v.map((d) => d.usAqi).sort(d3.ascending), 0.9)
				}),
				(d) => d3.timeMonth.floor(d.timestamp)
			),
			//sets the mean to the 15th of each month to give better spacing
			([date, data]) => ({
				date: new Date(date.getFullYear(), date.getMonth(), 15),
				mean: data.mean,
				firstQuantile: data.firstQuantile,
				lastQuantile: data.lastQuantile
			})
		);
	}

	//count the the number of records for each station and sort stations from most to least
	const stationRecordCount = $derived(
		Array.from(
			d3.rollup(
				data,
					(v) => v.length,
					(d) => d.stationName
				),
				([station, count]) => ({station, count})
			).sort((a, b) => d3.descending(a.count, b.count)
		)
	);

	let xScale = $derived(
		d3
			.scaleTime()
			.range([usableArea.left, usableArea.right])		
			.domain(d3.extent(selectedData, (d) => d.timestamp) as [Date, Date])
		);

	let yScale = $derived(
		d3
			.scaleLinear() 
			.domain([0, d3.max(selectedData, (d) => d.usAqi) ?? 0]) 
			.range([usableArea.bottom, usableArea.top]) 
	);

	//D3 line generator taking types, x value, and y value
	const line = $derived(
		d3
			.line<{date: Date; mean : number | undefined }>()
			.x((d) => xScale(d.date)) 
			.y((d) => yScale(d.mean ?? 0))
	);

	//D3 area generator taking types, x value, lower y value, and upper y value
	let area = $derived(
		d3
			.area<{ date: Date; firstQuantile: number | undefined; lastQuantile: number | undefined}>()
			.x((d) => xScale(d.date))
			.y0((d) => yScale(d.firstQuantile ?? 0))
			.y1((d) => yScale(d.lastQuantile ?? 0))
	);
	
	let xAxis = $derived(
		d3
			.axisBottom(xScale) //position on bottom of x axis
			.ticks(d3.timeYear.every(1)) //show 1 tick per year
			.tickFormat(d3.timeFormat('%Y') as any)); //format ticks as YYYY dates
	
	//positions y ticks on left of y axis
	let yAxis = $derived(
		d3
			.axisLeft(yScale));

	//positions the y grid lines to be the full width of the chart & removes labels
	let yAxisGridLines = $derived(
		d3
			.axisRight(yScale)
			.tickFormat(() => "")
			.tickSize(usableArea.right - usableArea.left));


	let xAxisRef: SVGGElement;
	let yAxisRef: SVGGElement;
	let yAxisGridLinesRef: SVGGElement;


	$effect(() => {
		if (xAxisRef && data.length > 0) {
			d3
				.select(xAxisRef)
				.call(xAxis)
		}
	});

	$effect(() => {
		if (yAxisRef && data.length > 0) {
			d3.select(yAxisRef)
			.call(yAxis);
		}
	});

	$effect(() => {
		if (yAxisGridLinesRef && data.length > 0) {
			d3.select(yAxisGridLinesRef)
			.call(yAxisGridLines);
		}
	})

</script>



<!-- dropdown menu that has an 'all' option showing the total data points, or options for each station. 
when a station is selected, it changes the value of 'thisStation' to be the selected station
when 'all selected' is seelcted, it changes the value of 'thisStation' to be null -->
<div class="menuItems">
	<select class="dropdownMenu" bind:value={thisStation}>
		<option value={null}>
			All Selected {data.length}
		</option>
		{#each stationRecordCount as station}
			<option value={station.station}>
				{station.station} ({station.count})
			</option>
		{/each}
	</select>

	<div class="checkbox"> 
		<p class="checkboxLabel">Show Raw Data</p>
		<input type="checkbox" bind:checked={dataPointsOn}/>
	</div>

	<!-- display of the record count for the currently selected station -->
	<p class="recordCount" >Number of Records: {selectedData.length}</p>
</div>


<svg {width} {height}>

	<!-- background colors as inidividual rectangles using the full x width and using the aqiLevel's min and max to set each rectangle's starting point and height
	only show the bands if they are within the y scale height -->
	{#each aqiLevels as aqiLevel}
		<rect
			x={usableArea.left}
			y={Math.max(usableArea.top, yScale(aqiLevel.max ?? 400))}
			width={usableArea.right - usableArea.left}
			height={Math.max(
				0,
				Math.min(yScale(aqiLevel.min), usableArea.bottom) -
				Math.max(usableArea.top, yScale(aqiLevel.max ?? 400))
			)}
			fill={aqiLevel.color}
			opacity="0.5"
		/>
	{/each}
	
	<!-- draw table elements in order so last drawn are in front -->
	<path class="shadedArea" d={area(findMonthData(selectedData))}/>
	<g class= "grid-lines" transform="translate({usableArea.left}, 0)" bind:this={yAxisGridLinesRef}></g>
	<g class="x-axis" transform="translate(0, {usableArea.bottom})" bind:this={xAxisRef}></g>
	<g class="y-axis" transform="translate({usableArea.left}, 0)" bind:this={yAxisRef}></g>
	
	<!-- show datapoints when checkbox is checked  -->
	{#if dataPointsOn}
		{#each selectedData as datapoints}
			<circle class="dataPoints"
					cx={xScale(datapoints.timestamp)}
					cy={yScale(datapoints.usAqi)}
					r=".85"
			/>
		{/each}
	{/if}
	
	<path class="stationMeanLine" d={line(findMonthData(selectedData))} />

</svg>


<style>

	* {
		font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', sans-serif;
		font-size: 10px;
	}

	.menuItems {
		margin-left: 40px;
		display: flex;
		flex-direction: column;
		align-items: left;
		justify-content: space-between;
		max-width: 20%;
	}

	.dropdownMenu {
		height: 20px;
		max-width: 200px;
		margin-top: 5px;
		margin-bottom: 5px;
	}

	.checkbox {
		display: flex;
		align-items: center;
		justify-content: left;
		height: 20px;
		padding-top: 5px;
	}

	.checkboxLabel {
		padding-right: 10px;
	}

	.recordCount {
		margin-top: 5px;

	}

	.shadedArea {
		color: black;
		opacity: .2;
	}
	.grid-lines {
		color: black;
		opacity: .1;
	}

	.stationMeanLine {
		stroke: black;
		fill: none;
		stroke-width: 1.5;
	}

	.dataPoints {
		fill: black;
	}
</style>


