<script lang="ts">
	import * as d3 from 'd3';

    // properties this component accepts
	const { data }: { data: Item[] } = $props();

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
	let height = $state(200);
	let margin = $state({top: 40, right: 20, bottom: 80, left: 40});

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
    
    let thisStation: string | null = $state("Pittsburgh");

    const selectedData = $derived(
		thisStation 
			? data.filter((d) => d.stationName === thisStation)
			: data
	);
    
    //find the mean AQI for each day in the year
    function findDailyAqi(data: Item[]) {
        return Array.from(
            d3.rollup(
                data,
                (v) => d3.mean(v, (d) => d.usAqi),
                (d) => `${d.timestamp.getMonth()}-${d.timestamp.getDate()}`
            ),
            ([date, mean]) => {
                let [month, day] = date.split('-').map(Number);
                //give december earlier year to set it in front of jan for seasonal organization
                if (month === 11) {
                    return {
                    date: new Date(1999, month, day),
                    mean
                }
                } else {
                    return {
                    date: new Date(2000, month, day),
                    mean
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

   	let xScale = $derived(
		d3
			.scaleTime()
			.range([usableArea.left, usableArea.right])		
			.domain(d3.extent(dailyAqi, (d) => d.date) as [Date, Date])
		);         

    
    let xAxis = $derived(
		d3
			.axisBottom(xScale) //position on bottom of x axis
            .tickValues(tickDates)
			.tickFormat(d3.timeFormat('%B') as any)); //format ticks as Months


    let xAxisRef: SVGGElement;

    $effect(() => {
		if (xAxisRef && data.length > 0) {
			d3
				.select(xAxisRef)
				.call(xAxis)
		}
	});

</script>


<svg {width} {height}>

    {#each seasonSort as day}
        <rect
            x={xScale(day.date)}
            y={usableArea.top}
            width={width / numberDays}
            height={(usableArea.bottom - usableArea.top)}
            fill={colorScale(day.mean ?? 0)}
        />
    {/each}

    <g class="x-axis" transform="translate(0, {usableArea.bottom})" bind:this={xAxisRef}></g>

</svg>




<style>

</style>
