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

    //visualizing traffic flow
    let stationFlow = d3.scaleQuantize()
        .domain([0, 1])
        .range([0, 0.5, 1]);

    //highlighting selected station
    let selectedStation = null;

    //adding isochrones
    const urlBase = 'https://api.mapbox.com/isochrone/v1/mapbox/';
    const profile = 'cycling';
    const minutes = [5, 10, 15, 20];
    const contourColors = [
        "03045e",
        "0077b6",
        "00b4d8",
        "90e0ef"
    ]
    let isochrone = null;

    async function getIso(lon, lat) {
        const base = `${urlBase}${profile}/${lon},${lat}`;
        const params = new URLSearchParams({
            contours_minutes: minutes.join(','),
            contours_colors: contourColors.join(','),
            polygons: 'true',
            access_token: mapboxgl.accessToken
        });
        const url = `${base}?${params.toString()}`;

        const query = await fetch(url, { method: 'GET' });
        isochrone = await query.json();
    }

    getIso(-71.09415, 42.36027);

    function geoJSONPolygonToPath(feature) {
        const path = d3.path();
        const rings = feature.geometry.coordinates;

        for (const ring of rings) {
            for (let i = 0; i < ring.length; i++) {
                const [lng, lat] = ring[i];
                const { x, y } = map.project([lng, lat]);
                if (i === 0) path.moveTo(x, y);
                else path.lineTo(x, y);
            }
            path.closePath();
        }
        return path.toString();
    }

    $: if (selectedStation) {
        getIso(+selectedStation.Long, +selectedStation.Lat);
    } else {
        isochrone = null;
    }



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
        {#key mapViewChanged}
            {#if isochrone}
                {#each isochrone.features as feature}
                    <path
                            d={geoJSONPolygonToPath(feature)}
                            fill={feature.properties.fillColor}
                            fill-opacity="0.2"
                            stroke="#000000"
                            stroke-opacity="0.5"
                            stroke-width="1"
                    >
                        <title>{feature.properties.contour} minutes of biking</title>
                    </path>
                {/each}
            {/if}
            {#each filteredStations as station}
                <circle
                    cx={getCoords(station).cx}
                    cy={getCoords(station).cy}
                    r={radiusScale(station.totalTraffic)}
                    fill="steelblue"
                    fill-opacity=0.6
                    stroke="white"
                    style="--departure-ratio: { stationFlow(station.departures / station.totalTraffic) }"
                    class={station?.Number === selectedStation?.Number ? "selected" : ""}
	                on:mousedown={() => selectedStation = selectedStation?.Number !== station?.Number ? station : null}
                >
                    <title>{station.totalTraffic} trips ({station.departures} departures, { station.arrivals} arrivals)</title>
                </circle>
            
            {/each}
        {/key}
    </svg>
</div>
<div class="legend">
    <div id="departure" style="--departure-ratio: 1">More departures</div>
    <div id="balanced" style="--departure-ratio: 0.5">Balanced</div>
    <div id="arrivals" style="--departure-ratio: 0">More arrivals</div>
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


    #map svg circle {
        pointer-events: auto;
        fill: var(--color);
        transition: opacity 0.2s ease;
    }

    /* Dimming effect when one circle is selected */
    #map svg:has(circle.selected) circle:not(.selected) {
        opacity: 0.3;
    }

    .legend {
        display: flex;
        margin-block: 1em;
        padding: 0;
        width: 100%;
        
    }

    .legend > div{
        background-color: var(--color);
        text-align: center;
        color: white;
        width: 100%;
        border-style: solid;
    }

    #map circle, .legend > div {
        --color-departures: steelblue;
        --color-arrivals: darkorange;
        --color: color-mix(
            in oklch,
            var(--color-departures) calc(100% * var(--departure-ratio)),
            var(--color-arrivals)
        ); 
        
    }

  

</style>