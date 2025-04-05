<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import { onMount } from "svelte";
    import * as d3 from 'd3';
    
    mapboxgl.accessToken = "pk.eyJ1IjoiYXV0dW1uYXJ0aXN0IiwiYSI6ImNtOTFtdTdvejAycTQyaW42aDA4M3duMGoifQ.ySZoxVyD5VLs_LUcDp3jPA";
    let stations = [];
    let trips = [];
    let departures, arrivals;
    let map;
    let mapViewChanged = 0;
    let timeFilter = -1;
    let stationFlow = d3.scaleQuantize()
        .domain([0, 1])
        .range([0, 0.5, 1]);

    function getCoords (station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let {x, y} = map.project(point);
        return {cx: x, cy: y};
    }
    function updateStationTraffic() {
        departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
        arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);
        
        // Merge traffic data into stations
        stations = stations.map(station => {
            let id = station.Number;
            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures;
            return station;
        });
    }
    function minutesSinceMidnight (date) {
        return date.getHours() * 60 + date.getMinutes();
    }
    
    async function initMap() {
        map = new mapboxgl.Map({
            container: 'map',
            center: [-71.09415, 42.36027],
            zoom: 12,
            style: "mapbox://styles/mapbox/streets-v12",
        });
        await new Promise(resolve => map.on("load", resolve));

        map.addSource("boston_route", {
            type: "geojson",
            data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
        });
        map.addLayer({
            id: "boston_bikes", // A name for our layer (up to you)
            type: "line", // one of the supported layer types, e.g. line, circle, etc.
            source: "boston_route", // The id we specified in `addSource()`
            paint: {
                // paint params, e.g. colors, thickness, etc.
                "line-color": "green",
                "line-width": 3,
                "line-opacity": 0.4
            },
        });
        map.addSource("cambridge_route", {
            type: "geojson",
            data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
        });
        map.addLayer({
            id: "cambridge_bikes", // A name for our layer (up to you)
            type: "line", // one of the supported layer types, e.g. line, circle, etc.
            source: "cambridge_route", // The id we specified in `addSource()`
            paint: {
                // paint params, e.g. colors, thickness, etc.
                "line-color": "green",
                "line-width": 3,
                "line-opacity": 0.4
            },
        });
        await initBikeStations();
        await initBikeTraffic();
        updateStationTraffic();
    }
    async function initBikeStations(){
        stations = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-stations.csv");
    }
    async function initBikeTraffic(){
        trips = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv")
        .then(rawTrips => {
            for (let trip of rawTrips) {
                trip.started_at = new Date(trip.started_at); 
                trip.ended_at   = new Date(trip.ended_at);
            }
            return rawTrips;
        });
    }

    $: radiusScale = d3.scaleSqrt()
            .domain([0, d3.max(stations, d => d.totalTraffic) || 0])
            .range([0, 25]);
    $: map?.on("move", evt => mapViewChanged++);
    $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
                     .toLocaleString("en", {timeStyle: "short"});
    $: filteredTrips = timeFilter === -1? trips : trips.filter(trip => {
        let startedMinutes = minutesSinceMidnight(trip.started_at);
        let endedMinutes = minutesSinceMidnight(trip.ended_at);
        return Math.abs(startedMinutes - timeFilter) <= 60
            || Math.abs(endedMinutes - timeFilter) <= 60;
    });
    $: filteredDepartures = d3.rollup(filteredTrips, v => v.length, d => d.start_station_id);
    $: filteredArrivals = d3.rollup(filteredTrips, v => v.length, d => d.end_station_id);
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


    onMount(() => {
        initMap();
    });
</script>

<header>
    <h1>🚲 BikeWatch</h1>
    <label>
        Filter by time:
        <input type="range" min="-1" max="1440" bind:value={timeFilter} />
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
            {#each filteredStations as station}
                <circle cx={ getCoords(station).cx }
                cy={ getCoords(station).cy }
                r={ radiusScale(station.totalTraffic) }
                style="--departure-ratio: { stationFlow(station.departures / station.totalTraffic) }"
                fill-opacity="0.60"
                stroke="white">
                    <title>{station.totalTraffic} trips ({station.departures} departures, { station.arrivals} arrivals)</title>
                </circle>
            {/each}
          {/key}
    </svg>
</div>
<div class="legend">
    <div style="--departure-ratio: 1">More departures</div>
    <div style="--departure-ratio: 0.5">Balanced</div>
    <div style="--departure-ratio: 0">More arrivals</div>
</div>


<style>
    @import url("$lib/global.css");
    #map {
	    flex: 1;
    }
    #map svg {
        /* background: yellow;
        opacity: 50%; */
        position: absolute;
        z-index: 1;
        width: 100%;
        height: 100%;
        pointer-events: auto;
    }
    /* circle {
        --color-departures: steelblue;
        --color-arrivals: darkorange;
        --color: color-mix(
            in oklch,
            var(--color-departures) calc(100% * var(--departure-ratio)),
            var(--color-arrivals)
        );
        fill: var(--color);
    } */
    #map circle,
    .legend > div {
        --color-departures: steelblue;
        --color-arrivals: darkorange;
        --color: color-mix(
            in oklch,
            var(--color-departures) calc(100% * var(--departure-ratio)),
            var(--color-arrivals)
        );
    }
    #map circle {
        fill: var(--color);
        fill-opacity: 0.6;
        stroke: white;
    }
    .legend > div {
        background: var(--color);
        color: black;
        padding: 0.5rem;
        /* border-radius: 4px; */
        border: 1px solid #ccc;
        text-align: center;
        width: 100%;
    }
    .legend {
        display: flex;
        gap: 1rem;
        margin-block: 1rem;
    }


</style>