<script lang="ts">
	import AQIChart from '$lib/AQIChart.svelte';
	import AQISecondVisual from '$lib/AQISecondVisual.svelte';
	import * as d3 from 'd3';

	const datasets = {
		avalon: 'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BAvalon%5D_daily-avg.csv',
		glassport_high_street:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BGlassport%20High%20Street%5D_daily-avg.csv',
		lawrenceville:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BLawrenceville%5D_daily-avg.csv',
		liberty_sahs:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BLiberty%20(SAHS)%5D_daily-avg.csv',
		manchester:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BManchester%5D_daily-avg.csv',
		north_braddock:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BNorth%20Braddock%5D_daily-avg.csv',
		parkway_east_near_road:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BParkway%20East%20(Near%20Road)%5D_daily-avg.csv',
		usa_pennsylvania_pittsburgh:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BUSA-Pennsylvania-Pittsburgh%5D_daily-avg.csv'
	};


	const data = $derived.by(() =>
		Promise.all(
			Object.values(datasets).map((url) =>
				d3.csv(url, (d: any) => ({
					city: d.City,
					country: d.Country,
					mainPollutant: d['Main pollutant'],
					pm25: +d['PM2.5'],
					state: d.State,
					stationName: d['Station name'] || 'Pittsburgh',
					timestamp: new Date(d['Timestamp(UTC)']),
					usAqi: +d['US AQI']
				}))
			)
		).then((allData) => allData.flat())
	);

	//variable to hold current station (or null if none are selected)
	//default is set to lawrenceville, the station shown in the assigment example
	let thisStation: string | null = $state("Lawrenceville");

	// ADD THIS:
	$effect(() => {
		console.log('Parent thisStation:', thisStation);
	});
	
</script>



{#await data}
	<!-- promise is pending -->
	<p>loading data...</p>
{:then data}
	<!-- promise was fulfilled or not a Promise -->
	<h2 class="title">AQI Chart</h2>

	<AQIChart {data} bind:thisStation/>

	<h2 class="title">AQI Chart 2</h2>

	<AQISecondVisual {data} bind:thisStation />

{:catch error}
	<!-- promise was rejected -->
	<p>Something went wrong: {error.message}</p>
{/await}


<style>
	* {
		font-family: sans-serif;
	}

	.title {
		padding-left: 40px;
	}

</style>
