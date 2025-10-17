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
	let height = $state(400);
	let margin = $state({top: 40, right: 20, bottom: 80, left: 40});

	let usableArea = $derived({
		top: margin.top,
		right: width - margin.right,
		bottom: height - margin.bottom,
		left: margin.left
	});

    //seasons
    const seasons: Array<string> = ['Winter', 'Spring', 'Summer', 'Fall'];

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
                return {
                    date: new Date(2000, month, day),
                    mean
                };
            }
        );
    }

    let dailyAqi = $derived(findDailyAqi(selectedData));

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

    //group the data by seasons - first compare seasons, then compare dates within that season
    //this does not change the original dailyAqi 
    const seasonGroup = $derived(
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

</script>



<h2>{thisStation}</h2>


{#each seasonGroup as day}
    {#if day.mean}
        <p>{day.date}: {day.mean}</p>
    {/if}
{/each}



<style>

</style>
