<script>
    import {onMount} from "svelte";
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";

    mapboxgl.accessToken = "pk.eyJ1IjoiYWthbXJhMTE4IiwiYSI6ImNtOTRrbHg3aTB6MWMyanB5bmh3c2dwMmYifQ.AUIoUyEPsIoAgVXIIIjmdg";

    onMount(() => {
        let map = new mapboxgl.Map({
            container: 'map',
            center:[-71.09415, 42.36027],
            minZoom: 10,
            maxZoom: 20,
            zoom: 12,
            style: "mapbox://styles/mapbox/light-v11",
        })
    })

</script>

<h1>🚲BikeBash🚲</h1>
<p>Visualizing bike traffic in Boston over time</p>

<div id="map">

</div>

<style>
    @import url("$lib/global.css");

    #map {
        flex: 1;
        background-color: blue
    }
</style>