<script>
    import {onMount} from "svelte";
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import * as d3 from "d3";

    let stations = [];
    let trips = [];
    let departures = new Map();
    let arrivals = new Map();
    let map;
    let mapViewChanged = 0;
    const departuresByMinute = Array.from({length: 1440}, () => []);
    const arrivalsByMinute = Array.from({length: 1440}, () => []);

    mapboxgl.accessToken = "pk.eyJ1IjoiYWthbXJhMTE4IiwiYSI6ImNtOTRrbHg3aTB6MWMyanB5bmh3c2dwMmYifQ.AUIoUyEPsIoAgVXIIIjmdg";

    async function load_map(){
        map = new mapboxgl.Map({
            container: 'map',
            center:[-71.09415, 42.36027],
            minZoom: 10,
            maxZoom: 20,
            zoom: 12,
            style: "mapbox://styles/mapbox/light-v11",
        })
        await new Promise(resolve => map.on("load", resolve));
        map.addSource("boston_route", {
            type:"geojson",
            data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
        });
        map.addSource("cambridge_route", {
            type:"geojson",
            data:"https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson"
        });
        map.addLayer({
            id: "boston_1", // A name for our layer (up to you)
            type: "line",
            source: "boston_route",
            paint: {
                "line-color": "#327fa8",
                "line-width": 0.8,
                "line-opacity": 0.5
                // paint params, e.g. colors, thickness, etc.
            },
        });
        map.addLayer({
            id: "boston_2", // A name for our layer (up to you)
            type: "line",
            source: "cambridge_route",
            paint: {
                "line-color": "#56a832",
                "line-width": 0.8,
                "line-opacity": 0.5
            },
        });
    }

    async function load_data(){
        stations = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-stations.csv");
        trips = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv").then(trips => {
            for (let trip of trips) {
                trip.started_at = new Date(trip.started_at);
                trip.ended_at = new Date(trip.ended_at);
                let startedMinutes = minutesSinceMidnight(trip.started_at);
                let endedMinutes = minutesSinceMidnight(trip.ended_at);

                departuresByMinute[startedMinutes].push(trip);
                arrivalsByMinute[endedMinutes].push(trip);
            }
            return trips;
        });
       

// TODO: Same for arrivals

    }

    function minutesSinceMidnight (date) {
        return date.getHours() * 60 + date.getMinutes();
    }

    function getCoords(station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let {x, y} = map.project(point);
        return {cx: x, cy: y};
    }

    onMount(async () => {
        await Promise.all([
            load_data(),
            load_map()
        ]); //makes us wait till the data is loaded before moving onto to defining departures and arrivals

        departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
        arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);
        stations = stations.map(station => {
            let id = station.Number;
            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures;
            return station;
        });
    })

    $: map?.on("move", evt=> mapViewChanged++);
    $: radiusScale = d3.scaleSqrt()
        .domain([0, d3.max(stations, d => d.totalTraffic) || 0])
        .range(timeFilter === -1 ? [0, 25] : [3,30]);

    let timeFilter = -1;
    $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
                     .toLocaleString("en", {timeStyle: "short"});

    
    function filterByMinute (tripsByMinute, minute) {
        // Normalize both to the [0, 1439] range
        // % is the remainder operator: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Remainder
        let minMinute = (minute - 60 + 1440) % 1440;
        let maxMinute = (minute + 60) % 1440;

        if (minMinute > maxMinute) {
            let beforeMidnight = tripsByMinute.slice(minMinute);
            let afterMidnight = tripsByMinute.slice(0, maxMinute);
            return beforeMidnight.concat(afterMidnight).flat();
        }
        else {
            return tripsByMinute.slice(minMinute, maxMinute).flat();
        }
    }
    $: filteredDepartures = d3.rollup(filterByMinute(departuresByMinute, timeFilter), v => v.length, d => d.start_station_id);
    $: filteredArrivals = d3.rollup(filterByMinute(arrivalsByMinute, timeFilter), v => v.length, d => d.end_station_id);

    $: filteredStations = stations.map(station => {
        const id = station.Number;
        const arr = filteredArrivals.get(id) ?? 0;
        const dep = filteredDepartures.get(id) ?? 0;
        return {
            ...station,
            arrivals: arr,
            departures: dep,
            totalTraffic: arr + dep
        };
    });


</script>


<header>
    <h1>🚲BikeBash🚲</h1>
    <label class="filter">
        Filter by Time
        <input bind:value={timeFilter} type="range" min="-1" max="1440">
        {#if timeFilter !== -1}
            <time style="display: block">
                {timeFilterLabel}
            </time>
        {:else}
            <em style="display: block">(any time)</em>
        {/if}
    </label>
   
</header>





<div id="map">
    <svg>
        {#each filteredStations as station}
            {#key mapViewChanged}
                <circle
                    cx={getCoords(station).cx}
                    cy={getCoords(station).cy}
                    r={radiusScale(station.totalTraffic)}
                    fill="steelblue"
                    fill-opacity=0.6
                    stroke="white"
                >
                    <title>{station.totalTraffic} trips ({station.departures} departures, { station.arrivals} arrivals)</title>
                </circle>
            {/key}
        {/each}
    </svg>
</div>

<style>
    @import url("$lib/global.css");

    #map {
        flex: 1;
        background-color: blue
    }

    #map svg {
        position: absolute;
        z-index: 1;
        width: 100%;
        height: 100%;
        pointer-events: none;
    }

    circle {
        pointer-events: auto;
    }


    header {
        display: flex;
        gap: 1em;
        align-items: baseline;
    }

    label {
        margin-left: auto;
    }

    em {
        font-style: italic;
    }

  

</style>