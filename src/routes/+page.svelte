<script>
  import mapboxgl from "mapbox-gl";
  import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
  mapboxgl.accessToken =
    "pk.eyJ1IjoiZ2xvcmlhemh1MDAiLCJhIjoiY205NmxqN2E4MW5kNzJscG5zdzVxMGRiMiJ9.bDMiiIwWttaq6wjReuvhEg";
  import * as d3 from "d3";
  import { onMount } from "svelte";

  let stations = [];
  let trips = [];
  let map;
  let mapViewChanged = 0;
  let departures = {};
  let arrivals = {};
  let timeFilter = -1;
  let selectedStation = null;

  async function initMap() {
    map = new mapboxgl.Map({
      container: "map",
      center: [-71.09415, 42.36027],
      zoom: 12,
      style: "mapbox://styles/mapbox/streets-v12",
    });
    await new Promise((resolve) => map.on("load", resolve));
    map.addSource("boston_route", {
      type: "geojson",
      data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
    });
    map.addSource("cambridge_route", {
      type: "geojson",
      data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
    });
    map.addLayer({
      id: "SOME_ID", // A name for our layer (up to you)
      type: "line", // one of the supported layer types, e.g. line, circle, etc.
      source: "boston_route", // The id we specified in `addSource()`
      paint: {
        // paint params, e.g. colors, thickness, etc.
        "line-width": 1.25,
        "line-opacity": 0.5,
        "line-color": "green",
      },
    });
    map.addLayer({
      id: "SOME_ID2", // A name for our layer (up to you)
      type: "line", // one of the supported layer types, e.g. line, circle, etc.
      source: "cambridge_route", // The id we specified in `addSource()`
      paint: {
        // paint params, e.g. colors, thickness, etc.
        "line-width": 1.25,
        "line-opacity": 0.5,
        "line-color": "blue",
      },
    });
    stations = await d3.csv(
      "https://vis-society.github.io/labs/8/data/bluebikes-stations.csv"
    );

    trips = await d3
      .csv(
        "https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv"
      )
      .then((trips) => {
        for (let trip of trips) {
          trip.started_at = new Date(trip.started_at);
          trip.ended_at = new Date(trip.ended_at);
        }
        return trips;
      });

    departures = d3.rollup(
      trips,
      (v) => v.length,
      (d) => d.start_station_id
    );
    arrivals = d3.rollup(
      trips,
      (v) => v.length,
      (d) => d.end_station_id
    );

    stations = stations.map((station) => {
      let id = station.Number;
      station.arrivals = arrivals.get(id) ?? 0;
      station.departures = departures.get(id) ?? 0;
      station.totalTraffic = station.arrivals + station.departures;
      return station;
    });
  }
  function minutesSinceMidnight(date) {
    return date.getHours() * 60 + date.getMinutes();
  }
  onMount(() => {
    initMap();
  });
  function getCoords(station) {
    let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
    let { x, y } = map.project(point);
    return { cx: x, cy: y };
  }
  let stationFlow = d3.scaleQuantize().domain([0, 1]).range([0, 0.5, 1]);
  $: filteredTrips =
    timeFilter === -1
      ? trips
      : trips.filter((trip) => {
          let startedMinutes = minutesSinceMidnight(trip.started_at);
          let endedMinutes = minutesSinceMidnight(trip.ended_at);
          return (
            Math.abs(startedMinutes - timeFilter) <= 60 ||
            Math.abs(endedMinutes - timeFilter) <= 60
          );
        });
  $: filteredDepartures = d3.rollup(
    filteredTrips,
    (v) => v.length,
    (d) => d.start_station_id
  );
  $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter).toLocaleString("en", {
    timeStyle: "short",
  });
  $: filteredArrivals = d3.rollup(
    filteredTrips,
    (v) => v.length,
    (d) => d.end_station_id
  );
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

  $: map?.on("move", (evt) => mapViewChanged++);
  $: radiusScale = d3
    .scaleSqrt()
    .domain([0, d3.max(stations, (d) => d.totalTraffic) || 0])
    .range(timeFilter == -1 ? [0, 25] : [3, 30]);

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

<h1>Bikewatching!</h1>
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
        <!-- svelte-ignore a11y-no-static-element-interactions -->
        <circle
          {...getCoords(station)}
          r={radiusScale(station.totalTraffic)}
          fill="steelblue"
          style="--departure-ratio: {stationFlow(
            station.departures / station.totalTraffic
          )}"
          class={station?.Number === selectedStation?.Number ? "selected" : ""}
          on:mousedown={() => {
            selectedStation =
              selectedStation?.Number !== station?.Number ? station : null;
            console.log("clicked?");
          }}
        />
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
  .legend > div {
    --color-departures: steelblue;
    --color-arrivals: darkorange;
    --color: color-mix(
      in oklch,
      var(--color-departures) calc(100% * var(--departure-ratio)),
      var(--color-arrivals)
    );
    background-color: var(--color);
    flex: 1;
    text-align: center;
    color: white;
  }
  .legend {
    display: flex;
    /* margin-block:  */
  }
  #map {
    flex: 1;
  }
  circle {
    opacity: 50%;
    --color-departures: steelblue;
    --color-arrivals: darkorange;
    --color: color-mix(
      in oklch,
      var(--color-departures) calc(100% * var(--departure-ratio)),
      var(--color-arrivals)
    );
    fill: var(--color);
    transition: opacity 0.2s ease; /* optional */
    pointer-events: auto;
  }
  #map svg {
    position: absolute;
    z-index: 1;
    width: 100%;
    height: 100%;
    pointer-events: none;
    &:has(circle.selected) circle:not(.selected) {
      opacity: 0.3;
    }
  }
  path {
        pointer-events: auto;
    }
</style>
