<script>
  import mapboxgl from "mapbox-gl";
  import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
  import { onMount } from "svelte";
  import * as d3 from "d3";

  // --- MAPBOX ACCESS TOKEN ---
  // Replace this string with your actual Mapbox token:
  mapboxgl.accessToken =
    "pk.eyJ1IjoiYXV0dW1uYXJ0aXN0IiwiYSI6ImNtOTFtdTdvejAycTQyaW42aDA4M3duMGoifQ.ySZoxVyD5VLs_LUcDp3jPA";

  // --- GLOBAL STATE ---
  let map;
  let stations = [];
  let allTrips = []; // If you want to handle "any time" (timeFilter = -1) via all trips
  let mapViewChanged = 0;
  let timeFilter = -1;
  let selectedStation = null;

  // --- BUCKET ARRAYS ---
  // Each of these arrays has 1440 sub-arrays (one per minute of the day).
  // We will push each trip into exactly one "departure bucket" and one "arrival bucket."
  const departuresByMinute = Array.from({ length: 1440 }, () => []);
  const arrivalsByMinute = Array.from({ length: 1440 }, () => []);

  // --- HELPER: Convert a Date to "minutes since midnight" ---
  function minutesSinceMidnight(date) {
    return date.getHours() * 60 + date.getMinutes();
  }

  // --- HELPER: Given a minute in [0..1439], collect trips within ±60 minutes,
  // accounting for wrap-around at midnight. ---
  function filterByMinute(tripsByMinute, minute) {
    const minMinute = (minute - 60 + 1440) % 1440;
    const maxMinute = (minute + 60) % 1440;

    // If min > max, it means we cross midnight, so we do two slices:
    // (1) from minMinute to the end, (2) from 0 to maxMinute
    if (minMinute > maxMinute) {
      let beforeMidnight = tripsByMinute.slice(minMinute);
      let afterMidnight = tripsByMinute.slice(0, maxMinute);
      // Flatten each part, then combine
      return beforeMidnight.concat(afterMidnight).flat();
    } else {
      // Otherwise, just do one slice from minMinute to maxMinute
      return tripsByMinute.slice(minMinute, maxMinute).flat();
    }
  }

  // --- LOAD BLUEBIKES STATIONS ---
  async function initBikeStations() {
    stations = await d3.csv(
      "https://vis-society.github.io/labs/8/data/bluebikes-stations.csv"
    );
  }

  // --- LOAD TRIPS AND FILL THE BUCKETS ---
  async function initBikeTraffic() {
    let rawTrips = await d3.csv(
      "https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv"
    );

    for (let trip of rawTrips) {
      trip.started_at = new Date(trip.started_at);
      trip.ended_at = new Date(trip.ended_at);

      let startedMinutes = minutesSinceMidnight(trip.started_at);
      let endedMinutes = minutesSinceMidnight(trip.ended_at);

      departuresByMinute[startedMinutes].push(trip);
      arrivalsByMinute[endedMinutes].push(trip);
    }

    // Keep a copy of all trips if needed for the "any time" scenario
    allTrips = rawTrips;
  }

  // --- MAP INITIALIZATION ---
  async function initMap() {
    map = new mapboxgl.Map({
      container: "map",
      center: [-71.09415, 42.36027],
      zoom: 12,
      style: "mapbox://styles/mapbox/streets-v12",
    });
    await new Promise((resolve) => map.on("load", resolve));

    // Example: add a couple of route layers
    map.addSource("boston_route", {
      type: "geojson",
      data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
    });
    map.addLayer({
      id: "boston_bikes",
      type: "line",
      source: "boston_route",
      paint: {
        "line-color": "green",
        "line-width": 3,
        "line-opacity": 0.4,
      },
    });

    map.addSource("cambridge_route", {
      type: "geojson",
      data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
    });
    map.addLayer({
      id: "cambridge_bikes",
      type: "line",
      source: "cambridge_route",
      paint: {
        "line-color": "green",
        "line-width": 3,
        "line-opacity": 0.4,
      },
    });

    // Now load station info and trip data
    await initBikeStations();
    await initBikeTraffic();
  }

  // We'll call initMap() in onMount so it runs once when the component loads.
  onMount(() => {
    initMap();
  });

  // --- ROLL UP DEPARTURES & ARRIVALS (derivations) ---
  // If timeFilter == -1, we show "any time" => use allTrips
  // Otherwise, pick from the relevant minute buckets
  $: filteredDepartures =
    timeFilter === -1
      ? d3.rollup(
          allTrips,
          (v) => v.length,
          (d) => d.start_station_id
        )
      : d3.rollup(
          filterByMinute(departuresByMinute, timeFilter),
          (v) => v.length,
          (d) => d.start_station_id
        );

  $: filteredArrivals =
    timeFilter === -1
      ? d3.rollup(
          allTrips,
          (v) => v.length,
          (d) => d.end_station_id
        )
      : d3.rollup(
          filterByMinute(arrivalsByMinute, timeFilter),
          (v) => v.length,
          (d) => d.end_station_id
        );

  // Now compute the "filteredStations" to show on the map’s overlay
  $: filteredStations = stations.map((station) => {
    const id = station.Number;
    const arr = filteredArrivals.get(id) ?? 0;
    const dep = filteredDepartures.get(id) ?? 0;
    return {
      ...station,
      arrivals: arr,
      departures: dep,
      totalTraffic: arr + dep,
    };
  });

  // The circle radius scale depends on the max totalTraffic among the filtered stations
  $: radiusScale = d3
    .scaleSqrt()
    .domain([0, d3.max(filteredStations, (d) => d.totalTraffic) || 0])
    .range([0, 25]);

  // Force re-draw of circles whenever the map moves
  $: map?.on("move", () => mapViewChanged++);

  // For convenience, a label that shows the timeFilter in a local time string
  $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter).toLocaleString("en", {
    timeStyle: "short",
  });

  // Simple helper to project lat/lon onto the map's screen coordinates
  function getCoords(station) {
    // Make sure that map is defined (i.e., after load).
    if (!map) return { cx: 0, cy: 0 };
    let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
    let { x, y } = map.project(point);
    return { cx: x, cy: y };
  }

  // Let's color stations proportionally to the ratio of departures vs. arrivals
  // e.g. 1 => all departures (blue), 0 => all arrivals (orange)
  function departureRatio(station) {
    if (station.totalTraffic === 0) return 0.5;
    return station.departures / station.totalTraffic;
  }

  // Requesting isochrones from mapbox
  const urlBase = "https://api.mapbox.com/isochrone/v1/mapbox/";
  const profile = "cycling";
  const minutes = [5, 10, 15, 20];
  const contourColors = ["03045e", "0077b6", "00b4d8", "90e0ef"];
  let isochrone = null;

  async function getIso(lon, lat) {
    const base = `${urlBase}${profile}/${lon},${lat}`;
    const params = new URLSearchParams({
      contours_minutes: minutes.join(","),
      contours_colors: contourColors.join(","),
      polygons: "true",
      access_token: mapboxgl.accessToken,
    });
    const url = `${base}?${params.toString()}`;

    const query = await fetch(url, { method: "GET" });
    isochrone = await query.json();
  }

  // Loading the isochrones
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
  <h1>🚲 BikeWatch</h1>
  <label>
    Filter by time:
    <input
      type="range"
      min="-1"
      max="1440"
      bind:value={timeFilter}
      style="width: 25%"
    />
    {#if timeFilter !== -1}
      <time style="display: block">{timeFilterLabel}</time>
    {:else}
      <em style="display: block">(any time)</em>
    {/if}
  </label>
</header>

<!-- MAP + SVG OVERLAY -->
<div id="map">
  <!-- We'll add an SVG overlay for station circles on top of the Mapbox map -->
  <svg>
    <!-- The #key ensures re-draw on mapViewChanged -->
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
          style="--departure-ratio: {departureRatio(station)}"
          fill-opacity="0.60"
          stroke="white"
          class={station?.Number === selectedStation?.Number ? "selected" : ""}
          on:mousedown={() =>
            (selectedStation =
              selectedStation?.Number !== station?.Number ? station : null)}
        >
          <title>
            {station.totalTraffic} trips ({station.departures} departures, {station.arrivals}
            arrivals)
          </title>
        </circle>
      {/each}
    {/key}
  </svg>
</div>

<!-- LEGEND -->
<div class="legend">
  <div style="--departure-ratio: 1">More departures</div>
  <div style="--departure-ratio: 0.5">Balanced</div>
  <div style="--departure-ratio: 0">More arrivals</div>
</div>

<style>
  /* Main layout */
  #map {
    position: relative;
    flex: 1;
    height: 600px; /* or however you want to size it */
  }

  #map svg {
    position: absolute;
    z-index: 1;
    width: 100%;
    height: 100%;
    pointer-events: none; /* So map is still clickable/pannable, except on circles themselves */
    &:has(circle.selected) circle:not(.selected) {
      opacity: 0.3;
    }
  }

  #map circle {
    pointer-events: auto; /* So we can click circles */
    fill: var(--color);
    fill-opacity: 0.6;
    stroke: white;
    transition: opacity 0.2s ease;
  }

  #map path {
    pointer-events: auto;
  }

  /* We define two custom CSS variables for color mixing:
       --color-departures: steelblue;
       --color-arrivals: darkorange;
  
       Then we mix them using color-mix() based on the ratio we store in --departure-ratio.
    */
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

  /* Legend styling */
  .legend {
    display: flex;
    gap: 1rem;
    margin-block: 1rem;
  }
  .legend > div {
    background: var(--color);
    color: black;
    padding: 0.5rem;
    border: 1px solid #ccc;
    text-align: center;
    width: 100%;
  }
</style>
