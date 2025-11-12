<template>
  <div ref="mapContainer" class="map-container"></div>
</template>

<script>
import mapboxgl from "mapbox-gl";
import "mapbox-gl/dist/mapbox-gl.css";


// API access token for Mapbox
mapboxgl.accessToken =
  "";

export default {
  props: ["modelValue"],

  data(){
    return{
        geojsonUrl: "/shipslogs_dummy.geojson"
    };
  },
  computed: {
    displayPosition() {
      const log = this.selectedLog;
      
      // Controleer of de log bestaat en de coördinaten numeriek zijn
      if (log && typeof log.longitude === 'number' && typeof log.latitude === 'number') {
        // Formatteer de coördinaten als string
        return `${log.longitude.toFixed(2)}, ${log.latitude.toFixed(2)}`;
      } 
      // Fallback voor ongedefinieerde/null coördinaten
      return 'Niet geregistreerd (bijv. aan land/haven)';
    }
  },

  mounted() {
    const { center, zoom } = this.modelValue

    // instantiate the map using the center and zoom from the modelValue prop
    const map = new mapboxgl.Map({
      container: this.$refs.mapContainer,
      style: "mapbox://styles/saraamelia/cmhvxq5ik000z01qx982c29gn",
      center: center,
      zoom,
    });


    map.on('load', this.addGeoJsonLayer);
    const updateLocation = () =>
        this.$emit("update:modelValue", this.getLocation());

    map.on("move", updateLocation);
    map.on("zoom", updateLocation);

    this.map = map;

  },

  // clean up the map instance when the component is unmounted
  unmounted() {
    this.map.remove();
    this.map = null;
  },
  
  // watch for external changes to the modelValue prop and update the map accordingly
  watch: {
    modelValue(next) {
      const curr = this.getLocation();

      // Only flyTo if any of the values have changed
      if (
        curr.center.lng !== next.center.lng ||
        curr.center.lat !== next.center.lat ||
        curr.zoom !== next.zoom
      ) {
        this.map.flyTo({
          center: next.center,
          zoom: next.zoom,
        });
      }
    },
  },
  
  methods: {
    getLocation() {
      return {
        center: this.map.getCenter(),
        zoom: this.map.getZoom(),
      };
    },

    async addGeoJsonLayer() {
      try {
        // 1. Load data - API call to fetch GeoJSON
        //const response = await fetch(this.geojsonUrl);
        //if (!response.ok) {
        //    throw new Error(`HTTP error! status: ${response.status}`);
        //}
        //const data = await response.json();


        //Convert data to Json!
        const response = await fetch(this.geojsonUrl);// Using local dummy data path for now
        const dataJson = await response.json();

        const data = dataJson; 

        const routeLines = this.generateRoutes(data.features);

        // 2. Add Source 
        this.map.addSource('ship-routes-source', {
          type: 'geojson',
          data: routeLines // GeoJSON FeatureCollection
        });

        // 3. Add layer
        this.map.addLayer(
        {
            id: 'ship-routes-lines',
            type: 'line',
            source: 'ship-routes-source',
            'minzoom': 2,
            layout: {
                'line-join': 'round',
                'line-cap': 'round'
            },
            paint: {
                'line-color': [
                    'match',
                    ['get', 'ship_name'],
                    'Albion', '#A08060',      // Oud brons/bruin
                    'De Zeefakkel', '#708090', // Oud staal/grijsblauw
                    '#B0C4DE'                 // Licht staalblauw voor andere schepen
                ],
                
                'line-width': 6, 
                
                'line-opacity': 0.9, 
            
                'line-dasharray': [3, 2], // [lengte van streep, lengte van spatie]
            }
        });
        
        this.map.addSource('ship-log-source', {
            type: 'geojson',
            data: data
        });

    this.map.addLayer(
        {
          id: 'ship-log-points',
          type: 'circle',
          source: 'ship-log-source', // Reference to map source
          paint: {
            // Mapbox expression for style
            'circle-color': [
                'match',
                ['get', 'ship_name'],
                'De Zeefakkel', '#007bff',      // Blue
                'De Walvisvaarder', '#dc3545', // Red
                '#ccc'
            ],
            'circle-radius': 8,
            'circle-stroke-width': 8,
            'circle-stroke-color': '#fff'
          }
        },
        
    );

        //Interactivtity: mousehovers and popups
        this.addMapInteractivity();

      } catch (error) {
        console.error("Issue loading GEOJson:", error);
      }
    },

    addMapInteractivity() {
        this.map.on('click', 'ship-log-points', (e) => {
        const clickedFeature = e.features[0];
        
        // ZEND HET EVENT UIT: 'open-sidebar' met de feature properties
        this.$emit('open-sidebar', {
            ...clickedFeature.properties,
            longitude: clickedFeature.geometry.coordinates[0], 
            latitude: clickedFeature.geometry.coordinates[1]
        });

        // Optioneel: centreer de kaart op het geklikte punt
        this.map.flyTo({
            center: clickedFeature.geometry.coordinates,
            speed: 0.5,
            essential: true
        });
    });
        
        // Change cursor at hover
        this.map.on('mouseenter', 'ship-log-points', () => {
            this.map.getCanvas().style.cursor = 'pointer';
        });
        this.map.on('mouseleave', 'ship-log-points', () => {
            this.map.getCanvas().style.cursor = '';
        });
    },

    // Generate routes
    generateRoutes(pointFeatures){
        const routes = {};

        //Group and sort points
        pointFeatures.forEach(feature => {
            const log_id = feature.properties.log_id;
            if (!routes[log_id]) {
                routes[log_id] = [];
            }
            routes[log_id].push(feature);
        });

        const routeFeatures = [];
        for (const log_id in routes) {
        // Sorteer punten op datum (van oud naar nieuw)
        routes[log_id].sort((a, b) => new Date(a.properties.date) - new Date(b.properties.date));

        // Extraheer de gesorteerde coördinaten
        const coordinates = routes[log_id].map(f => f.geometry.coordinates);

        // 2. Maak de LineString Feature
        if (coordinates.length > 1) { 
            // Vroegste en laatste datum (voor pop-ups op de lijn)
            const startDate = routes[log_id][0].properties.date;
            const endDate = routes[log_id][routes[log_id].length - 1].properties.date;

            routeFeatures.push({
                type: 'Feature',
                geometry: {
                    type: 'LineString',
                    coordinates: coordinates
                },
                properties: {
                    ship_name: routes[log_id][0].properties.ship_name,
                    voyage_id: log_id,
                    period: `${startDate} tot ${endDate}`
                }
            });
        }
    }
    return {
        type: 'FeatureCollection',
        features: routeFeatures
    };
    }
  },
};

</script>

<style>
.map-container {
  width: 100%;
  height: 100%;
}
</style>
