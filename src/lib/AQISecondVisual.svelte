<script lang="ts">
	import * as d3 from 'd3';

    // properties this component accepts
	let { data, thisStation = $bindable() }: { data: Item[], thisStation: string | null } = $props();

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

    // chart dimension variables
	let width = $state(700);
	let height = $state(150);
	let margin = $state({top: 15, right: 40, bottom: 20, left: 40});

    //legend dimension variables
    let legWidth = $state(175);
    let legHeight = $state(20);

	let usableArea = $derived({
		top: margin.top,
		right: width - margin.right,
		bottom: height - margin.bottom,
		left: margin.left
	});

    //seasons
    const seasons: Array<string> = ['Winter', 'Spring', 'Summer', 'Fall'];

    //days (for x axis label orientation)
    const tickDates: Array<Date> = [
        new Date(1999, 11, 15),
        new Date(2000, 0, 15),
        new Date(2000, 1, 15),
        new Date(2000, 2, 15),
        new Date(2000, 3, 15),
        new Date(2000, 4, 15),
        new Date(2000, 5, 15),
        new Date(2000, 6, 15),
        new Date(2000, 7, 15),
        new Date(2000, 8, 15),
        new Date(2000, 9, 15),
        new Date(2000, 10, 15)
    ];

    //Colored AQI definitions
	const aqiLevels = [
		{ name: 'Good', min: 0, max: 50, color: '#9cd84e' },
		{ name: 'Moderate', min: 50, max: 100, color: '#facf39' },
		{ name: 'Unhealthy for Sensitive Groups', min: 100, max: 150, color: '#f99049' },
		{ name: 'Unhealthy', min: 150, max: 200, color: '#f65e5f' },
		{ name: 'Very Unhealthy', min: 200, max: 300, color: '#a070b6' },
		{ name: 'Hazardous', min: 300, color: '#a06a7b' }
	];
    
    // let thisStation: string | null = $state("Pittsburgh");

    const selectedData = $derived(
		thisStation 
			? data.filter((d) => d.stationName === thisStation)
			: data
	);

    //min-max year calculations for informing viewer about the data range
    let firstYear = $derived(d3.min(selectedData, (d) => d.timestamp.getFullYear()));
    let lastYear = $derived(d3.max(selectedData, (d) => d.timestamp.getFullYear()));

    let selectedCalc = $state('mean');

    const calcList = [
        {calc: 'mean', name: 'Mean AQI'},
        {calc: 'maxAqi', name: 'Max AQI'}
    ];
    
    //find the mean AQI for each day in the year
    function findDailyAqi(data: Item[]) {
        return Array.from(
            d3.rollup(
                data,
                (v) => ({
                    mean: d3.mean(v, (d) => d.usAqi),
                    maxAqi: d3.max(v, (d) => d.usAqi),
                    minAqi: d3.min(v, (d) => d.usAqi),
                    //store the year of the maxAQI and minAQI but don't use it to sort
                    maxAqiYear: d3.greatest(v, (d) => d.usAqi)?.timestamp.getFullYear(),
                    minAqiYear: d3.least(v, (d) => d.usAqi)?.timestamp.getFullYear()
                }),
                (d) => `${d.timestamp.getMonth()}-${d.timestamp.getDate()}`
            ),
            ([date, data]) => {
                let [month, day] = date.split('-').map(Number);
                //give december earlier year to set it in front of jan for seasonal organization
                if (month === 11) {
                    return {
                    date: new Date(1999, month, day),
                    mean: data.mean,
                    maxAqi: data.maxAqi,
                    minAqi: data.minAqi,
                    maxAqiYear: data.maxAqiYear,
                    minAqiYear: data.minAqiYear
                }
                } else {
                    return {
                    date: new Date(2000, month, day),
                    mean: data.mean,
                    maxAqi: data.maxAqi,
                    minAqi: data.minAqi,
                    maxAqiYear: data.maxAqiYear,
                    minAqiYear: data.minAqiYear
                    };
                }
            }
        );
    }

    let dailyAqi = $derived(findDailyAqi(selectedData));

    //hold number of days for shape generator
    let numberDays = $derived(d3.count(dailyAqi, (d) => d.date.getTime()));
    
    //find the season of each date approximated by month, not by solstice/equinox date
    function getSeason(date: Date) {
        //find the month and add 1 because it's 0-11 which is confusing for months
        let month = (date.getMonth()+ 1);

        if (month > 2 && month <= 5) { //march to may
            return seasons[1];
        } else if (month > 5 && month <= 8) { //june to august
            return seasons[2];
        } else if (month > 8 && month <= 11) { //september to november
            return seasons[3];
        } else { //dec to feb
            return seasons[0]; 
        }
    }

    //Sort the data by seasons - first compare seasons, then compare dates within that season
    //does this change original dailyAqi??
    const seasonSort = $derived(
        Array.from(dailyAqi) 
            .sort((a, b) => {
                const season = d3.ascending(seasons.indexOf(getSeason(a.date)), seasons.indexOf(getSeason(b.date)));
                if (season !== 0) { 
	                return season; 
                } else {
                    let monthA = a.date.getMonth()+1;
                    let monthB = b.date.getMonth()+1;

                    //controlling for dec being in winter but mo. 12
                    if (monthA == 12 && (monthB === 1 || monthB === 2)) { //if month a is dec. and month b is jan or feb, dec goes first
                        return -1;
                    } else if (monthB == 12 && (monthA === 1 || monthA === 2)) { //opposite for month b being dec
                        return 1;
                    } else {
                        let month = d3.ascending(a.date.getMonth()+1, b.date.getMonth()+1); //otherwise sort months normally in each season
                        if (month !==0 ) {
                            return month;
                        }
                    }
                    return d3.ascending(a.date.getTime(), b.date.getTime());
                }
            })
    );

    let colorScale = $derived(
		d3
			.scaleThreshold<number , string>()
			.domain(
                aqiLevels
                    .filter((d) => d.max !== undefined)
                    .map((d) => d.max))
			.range(aqiLevels.map((d) => d.color))
	);

    let seasonLabelScale = $derived(
        d3
            .scaleBand()
            .range([usableArea.left, usableArea.right])
            .domain(seasons)
    );

   	let xScale = $derived(
		d3
			.scaleTime()
			.range([usableArea.left, usableArea.right])		
			.domain(d3.extent(dailyAqi, (d) => d.date) as [Date, Date])
    );         

    
    let xAxis = $derived(
		d3
			.axisTop(xScale) //position on bottom of x axis
            .tickValues(tickDates)
			.tickFormat(d3.timeFormat('%B') as any) //format ticks as Months
            .tickSize(0)
            .tickPadding(6)
    );

    let xAxisRef: SVGGElement;

    $effect(() => {
		if (xAxisRef && data.length > 0) {
			d3
				.select(xAxisRef)
				.call(xAxis)
		}
	});



</script>


<div class="visual">
    <div class='chart'>
        <svg {width} height="30px">
            {#each seasons as season}
                <rect class='seasonRect'
                    x={seasonLabelScale(season)}
                    y=0
                    width={seasonLabelScale.bandwidth()+2}
                    height="26px"
                />
                <text
                    class='seasonLabel'
                    x={(seasonLabelScale(season) ?? 0) + seasonLabelScale.bandwidth() / 2}
                    y="13"
                    dy="0.35em"
                >
                    {season}
                </text>
            {/each}
        </svg>

        <svg {width} {height}>
            {#each seasonSort as day}
                <rect class="allDays"
                    x={xScale(day.date)}
                    y={usableArea.top}
                    width={width / numberDays}
                    height={(usableArea.bottom - usableArea.top)}
                    fill={day[selectedCalc as keyof typeof day] !== undefined ? colorScale(day[selectedCalc as keyof typeof day] as number) : 'lightgrey'}
                />
            {/each}
            <g class="x-axis" transform="translate(0, {usableArea.top})" bind:this={xAxisRef}></g>
        </svg>
        
        <p class="chartTitle">Seasonal AQI Trends for {thisStation !== null ? (thisStation + " Station") : "All Stations"} in Pennsylvania, using data collected from {firstYear} to {lastYear}</p>
    </div>

    <div class="sidebar">
        <div class="dataSelection">
            <h4 class="radioMenuTitle">Select Data Filter:</h4>
            {#each calcList as calc}
                <label class="radioLabel">
                    <input class="radio" type="radio" bind:group={selectedCalc} value={calc.calc} />
                    {calc.name}
                </label>
            {/each}
        </div>

        <div class="legend">
            <p class="legendTitle">Legend</p>
            
            <svg>
                {#each aqiLevels as aqi, index}
                    <rect class="legendRect"
                        y={index * (legHeight+2)}
                        width={legWidth}
                        height={legHeight}
                        fill={aqi.color}
                    />
                    <text 
                        class="legendText"
                        x={10}
                        y={index * (legHeight+2) + (legHeight/2)}
                        dy="0.35em"
                    >
                        {aqi.name}
                    </text>
                {/each}
            </svg>
        </div>
    </div>
    

</div>






<style>

* {
    font-family: sans-serif;
}

.visual {
    display: flex;
    justify-content: top;
    align-items: center; 
}

.chartTitle {
    width: 700px;
    margin: 0px;
    font-size: 12px;
    text-align: center;
    font-weight: bold;
}

.sidebar {
    display: flex;
    flex-direction: column;
    align-items: left;
    justify-content:space-between;
    border-style: solid;
    border-color: lightgrey;
    padding: 10px;
    margin-left: 30px;
    width: 180px;
    height: 240px;
}

.dataSelection {
    font-size: 10px;
    padding-bottom: 10px;
}

.radioMenuTitle {
    margin: 0px 0px 5px 0px;
}

.radioLabel {
    display: flex;
    align-items: center;
    justify-content: left;
    padding: 5px 0px 0px 0px;
}

.radio {
    margin-right: 10px;
    margin-left: 0px;
}

.chart {
    padding-top: 10px;
    display: flex;
    flex-direction: column;
    justify-content:top;
    width: 700px;
    height: 275px;
}

.seasonRect {
    fill: lightgray;
    stroke: white;
    stroke-width: 2px;
}

.seasonLabel {
    text-anchor: middle;
    font-size: 12px;
}

.legendText {
    font-size: 10px;
}

.allDays {
    opacity: .7;
}

.x-axis {
    stroke-width: 0;
}

.legend {
    display: flex;
    flex-direction: column;
    justify-content: center;
    width: 180px;

}

.legendTitle {
    font-size: 10px;
    font-weight: bold;
}

.legendRect {
    opacity: .7;
}


</style>
