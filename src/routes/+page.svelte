<script>
    import {onMount} from "svelte";
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import * as d3 from "d3";

    let stations = [];
    let map;
    let mapViewChanged = 0;
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
                // paint params, e.g. colors, thickness, etc.
            },
        });
    }

    async function load_data(){
        stations = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-stations.csv");
    }

    function getCoords(station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let {x, y} = map.project(point);
        return {cx: x, cy: y};
    }


    onMount(() => {
        load_data();
        load_map();
    })

    $: map?.on("move", evt=> mapViewChanged++);

</script>

<h1>🚲BikeBash🚲</h1>
<p>Visualizing bike traffic in Boston over time</p>

<div id="map">
    <svg>
        {#each stations as station}
            {#key mapViewChanged}
                <circle
                    {...getCoords(station)}
                    r="3"
                    fill="steelblue"
                />
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

</style>