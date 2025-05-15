<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import * as d3 from "d3";

    mapboxgl.accessToken = "pk.eyJ1IjoidmluYXNxdWUiLCJhIjoiY21hcGg3a3Z6MGo0MDJqcG80ZDJwN2xjayJ9.WIJ57FQeu0Xmv-hiNb1QgA";

    import { onMount } from "svelte";

    let map;
    let stations = [];
    let mapViewChanged = 0;

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
                "line-color": "rgba(0, 255, 0, 0.5)", // verde
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
                "line-color": "rgba(0, 0, 255, 0.5)",  // azul
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
            // console.log('Stations loaded:', stations.length);
        } catch (error) {
            console.error('Error loading station data:', error);
        }
    }

    function getCoords (station) {
        let point = new mapboxgl.LngLat(station.Long, station.Lat);
        let {x, y} = map.project(point);
        return {cx: x, cy: y};
    }

    $: map?.on("move", evt => mapViewChanged++);

    async function loadStationDemand() {
        try {
            const csvUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv';
            const data = await d3.csv(csvUrl);
            
            trips = data.map(trip => {
                return {
                    id: trip.ride_id,
                    name: trip.NAME,
                    started_at: new Date(trip.started_at),
                    ended_at: new Date(trip.ended_at),
                    start_station_id: trip.start_station_id,
                    end_station_id: trip.end_station_id
                };
            });
            
            // console.log('Trips loaded:', trips.length);
        } catch (error) {
            console.error('Error loading station data:', error);
        }
    }

    onMount(() => {
        initMap();
        loadStationData();
        loadStationDemand();
    });
</script>

<h1>Bikes</h1>

<div id="map">
    {#key mapViewChanged}
        <svg>
            {#each stations as station}
                <circle cx={ getCoords(station).cx } cy={ getCoords(station).cy } r="5" fill="steelblue" />
            {/each}
        </svg>
    {/key}
</div>


<style> 
    @import url("$lib/global.css");

    #map {
        flex: 1;
    }

    #map svg {
        z-index: 1;
        position: absolute;
        width: 100%;
        height: 100%;
        pointer-events: none
    }
</style>