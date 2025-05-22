<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import * as d3 from "d3";
    import { onMount } from "svelte";

    mapboxgl.accessToken = "pk.eyJ1IjoidmluYXNxdWUiLCJhIjoiY21hcGg3a3Z6MGo0MDJqcG80ZDJwN2xjayJ9.WIJ57FQeu0Xmv-hiNb1QgA";

    let map;
    let stations = [];
    let mapViewChanged = 0;
    let departures = new Map();
    let arrivals = new Map();
    let trips = [];

    let timeFilter = -1;

    $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
                         .toLocaleString("en", { timeStyle: "short" });

    function minutesSinceMidnight(date) {
        return date.getHours() * 60 + date.getMinutes();
    }

    $: filteredTrips = timeFilter === -1 ? trips : trips.filter(trip => {
        let startedMinutes = minutesSinceMidnight(trip.started_at);
        let endedMinutes = minutesSinceMidnight(trip.ended_at);
        return Math.abs(startedMinutes - timeFilter) <= 60
            || Math.abs(endedMinutes - timeFilter) <= 60;
    });

    $: filteredDepartures = d3.rollup(filteredTrips, v => v.length, d => d.start_station_id);
    $: filteredArrivals = d3.rollup(filteredTrips, v => v.length, d => d.end_station_id);

    $: filteredStations = stations.map(station => {
        const id = station.id;
        const arr = filteredArrivals.get(id) ?? 0;
        const dep = filteredDepartures.get(id) ?? 0;
        return {
            ...station,
            arrivals: arr,
            departures: dep,
            totalTraffic: arr + dep
        };
    });

    $: radiusScale = d3.scaleSqrt()
        .domain([0, d3.max(filteredStations, d => d.totalTraffic) || 0])
        .range([0, 25]);

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
            id: "boston_bike_lanes",
            type: "line",
            source: "boston_route",
            paint: {
                "line-color": "rgba(0, 255, 0, 0.5)",
                "line-width": 3,
                "line-opacity": 0.9,
            },
        });

        map.addSource("cambridge_route", {
            type: "geojson",
            data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
        });

        map.addLayer({
            id: "cambridge_bike_lanes",
            type: "line",
            source: "cambridge_route",
            paint: {
                "line-color": "rgba(0, 0, 255, 0.5)",
                "line-width": 3,
                "line-opacity": 0.9,
            },
        });
    }

    async function loadStationData() {
        try {
            const csvUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-stations.csv';
            const data = await d3.csv(csvUrl);
            
            stations = data.map(station => ({
                id: station.Number,
                name: station.NAME,
                Lat: +station.Lat,
                Long: +station.Long,
            }));
        } catch (error) {
            console.error('Error loading station data:', error);
        }
    }

    function getCoords(station) {
        let point = new mapboxgl.LngLat(station.Long, station.Lat);
        let { x, y } = map.project(point);
        return { cx: x, cy: y };
    }

    $: map?.on("move", () => mapViewChanged++);

    async function loadStationDemand() {
        try {
            const csvUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv';
            trips = await d3.csv(csvUrl).then(trips => {
                for (let trip of trips) {
                    trip.started_at = new Date(trip.started_at);
                    trip.ended_at = new Date(trip.ended_at);
                }
                return trips;
            });

            departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
            arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);

            stations = stations.map(station => {
                let id = station.id;
                station.arrivals = arrivals.get(id) ?? 0;
                station.departures = departures.get(id) ?? 0;
                station.totalTraffic = station.arrivals + station.departures;
                return station;
            });
        } catch (error) {
            console.error('Error loading demand data:', error);
        }
    }

    onMount(() => {
        initMap();
        loadStationData();
        loadStationDemand();
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
    {#key mapViewChanged}
        <svg>
            {#each filteredStations as station}
                <circle 
                    cx={getCoords(station).cx} 
                    cy={getCoords(station).cy} 
                    r={radiusScale(station.totalTraffic)} 
                    fill="steelblue" />
            {/each}
        </svg>
    {/key}
</div>

<style>
    @import url("$lib/global.css");

    header {
        display: flex;
        gap: 1em;
        align-items: baseline;
    }

    label {
        margin-left: auto;
    }

    em {
        color: #888;
        font-style: italic;
    }

    #map {
        flex: 1;
        position: relative;
    }

    #map svg {
        z-index: 1;
        position: absolute;
        width: 100%;
        height: 100%;
        pointer-events: none;
    }

    #map svg circle {
        fill-opacity: 0.6;
        stroke: white;
        stroke-width: 1;
    }
</style>
