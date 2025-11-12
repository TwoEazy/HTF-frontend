<template>
  <div ref="mapContainer" class="map-container"></div>
</template>

<script>
import mapboxgl from "mapbox-gl";
import "mapbox-gl/dist/mapbox-gl.css";


// API access token for Mapbox
mapboxgl.accessToken =
  "pk.eyJ1Ijoic2FyYWFtZWxpYSIsImEiOiJjbWh2cmJ1eXowYjBjMmxzN3ltNm9iaG10In0.8RJ6ewgf0MR4vOWU5-WRQA";

export default {
  props: ["modelValue"],

  data(){
    return{
        geojsonUrl: "http://10.7.0.176:5000/api/routes"
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

    getHistoricColorPalette() {
    return [
   '#FF4500', // Orange Red (Zeer Helder Oranje-Rood)
        '#00BFFF', // Deep Sky Blue (Levendig, Helder Blauw)
        '#32CD32', // Lime Green (Fel Groen)
        '#FFD700', // Gold (Klassiek Helder Goud)
        '#FF1493', // Deep Pink (Opvallend Roze)
        '#8A2BE2', // Blue Violet (Intens Paars)
        '#1E90FF'  // Dodger Blue (Briljant Blauw)
    ];
},
createColorMapping(shipList) {
    const palette = this.getHistoricColorPalette();
    const colorMap = {};
    
    shipList.forEach((ship, index) => {
        // Gebruik de modulo operator (%) om door het palet te loopen als er meer schepen zijn dan kleuren
        const color = palette[index % palette.length];
        colorMap[ship.name] = color;
    });
    
    return colorMap;
},
generateColorExpression(colorMap) {
    // Start met de basis van de Mapbox match expressie
    let expression = [
        'match', 
        ['get', 'ship_name'] // De property om te matchen
    ];

    // Voeg voor elk schip de naam en de kleur toe
    for (const name in colorMap) {
        expression.push(name, colorMap[name]);
    }

    // Voeg de fallback kleur toe voor niet-gematched schepen
    expression.push('#696969'); // Donkergrijs als fallback

    return expression;
},


    async addGeoJsonLayer() {
      try {
        // 1. Load data - API call to fetch GeoJSON or API-structured JSON
        const response = await fetch(this.geojsonUrl);
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }

        const apiData = await response.json();

        // Convert API format into a GeoJSON FeatureCollection if necessary.
        // Expected API example (single object or array of objects):
        // { log_id, ship_name, coordinates: [{date, lat, lng, ...}, ...], ... }
        let geojson;
        if (apiData && apiData.type === 'FeatureCollection' && Array.isArray(apiData.features)) {
          geojson = apiData;
        } else {
          const records = Array.isArray(apiData) ? apiData : [apiData];
          const features = [];
          records.forEach(record => {
            if (!record || !Array.isArray(record.coordinates)) return;
            record.coordinates.forEach(coord => {
              const lat = Number(coord.lat ?? coord.latitude ?? coord.latitud ?? 0);
              const lng = Number(coord.lng ?? coord.longitude ?? coord.long ?? 0);
              if (Number.isFinite(lat) && Number.isFinite(lng)) {
                features.push({
                  type: 'Feature',
                  geometry: { type: 'Point', coordinates: [lng, lat] },
                  properties: {
                    ship_name: record.ship_name || record.master || 'unknown',
                    log_id: record.log_id || record.voyage_id || null,
                    date: coord.date || coord.datetime || null,
                    remarks: coord.remarks || null,
                    weather: coord.weather || null,
                    events: coord.events || [],
                    destination: record.destination || null,
                    master: record.master || null,
                    voyage_start: record.voyage_start || null,
                    voyage_end: record.voyage_end || null
                  }
                });
              }
            });
          });

          geojson = { type: 'FeatureCollection', features };
        }

        // Generate route lines from point features
        const routeLines = this.generateRoutes(geojson.features);
        let shipList = this.extractShipList(geojson.features);
        const colorMap = this.createColorMapping(shipList);
const dynamicColorExpression = this.generateColorExpression(colorMap);
this.$emit('data-loaded', shipList);

        // Build ships list with earliest (start) point per voyage/log
        const shipsMap = {};
        geojson.features.forEach(f => {
          const props = f.properties || {};
          const id = props.log_id || props.voyage_id || props.ship_name || 'unknown';
          const date = props.date ? new Date(props.date) : null;
          if (!shipsMap[id]) shipsMap[id] = { ship_name: props.ship_name || 'unknown', log_id: props.log_id || null, startFeature: null, startDate: null };
          const current = shipsMap[id];
          if (!current.startFeature) {
            current.startFeature = f;
            current.startDate = date;
          } else if (date && current.startDate && date < current.startDate) {
            current.startFeature = f;
            current.startDate = date;
          } else if (date && !current.startDate) {
            current.startFeature = f;
            current.startDate = date;
          }
        });

        const shipsList = Object.keys(shipsMap).map(k => {
          const s = shipsMap[k];
          const coords = s.startFeature ? s.startFeature.geometry.coordinates : null;
          return {
            ship_name: s.ship_name,
            log_id: s.log_id,
            start: coords,
            startDate: s.startDate ? s.startDate.toISOString() : null
          };
        });

        // Emit ships list so parent can build a menu
        this.$emit('ships-loaded', shipsList);

        // Add or update ship-routes-source
        if (this.map.getSource('ship-routes-source')) {
          this.map.getSource('ship-routes-source').setData(routeLines);
        } else {
          this.map.addSource('ship-routes-source', {
            type: 'geojson',
            data: routeLines
          });

          // Add routes layer if not present
          if (!this.map.getLayer('ship-routes-lines')) {
            this.map.addLayer({
              id: 'ship-routes-lines',
              type: 'line',
              source: 'ship-routes-source',
              minzoom: 2,
              layout: { 'line-join': 'round', 'line-cap': 'round' },
              paint: {
                'line-color': dynamicColorExpression,
                'line-width': 3,
                'line-opacity': 0.9,
                'line-dasharray': [1, 1.5]
              }
            });
          }
        }

        // Add or update ship-log-source (points)
        if (this.map.getSource('ship-log-source')) {
          this.map.getSource('ship-log-source').setData(geojson);
        } else {
          this.map.addSource('ship-log-source', { type: 'geojson', data: geojson });

          if (!this.map.getLayer('ship-log-points')) {
            this.map.addLayer({
              id: 'ship-log-points',
              type: 'circle',
              source: 'ship-log-source',
              paint: {
                'circle-radius': 6, 
        'circle-color': dynamicColorExpression, // <-- GEBRUIK DE DYNAMISCHE EXPRESSIE
        'circle-stroke-width': 1.5,
        'circle-stroke-color': '#F0E6D0',
        'circle-opacity': 0.8
              }
            });
          }
        }

        //Interactivtity: mousehovers and popups
        this.addMapInteractivity();

  // expose geojson on component instance for debugging if needed
  this._latestGeojson = geojson;

      } catch (error) {
        console.error("Issue loading GEOJson:", error);
      }
    },

    extractShipList(features) {
        console.log(features);
    const shipMap = {};

    // 1. Groepeer Features en bepaal de vroegste logboekdatum
    features.forEach(feature => {
        const properties = feature.properties;
        const coordinates = feature.geometry.coordinates;
        
        // Gebruik voyage_id als unieke sleutel
        const log_id = properties.log_id; 
        
    
        const currentDate = properties.date; 

        if (!shipMap[log_id]) {
            // Dit is het eerste punt dat we zien: beschouw dit als de start
            shipMap[log_id] = {
                name: properties.ship_name,
                log_id: log_id,
                
                // De vroegste logboekdatum gevonden
                startDate: currentDate, 
                
                // De algemene reisstart (indien beschikbaar in properties)
                voyageStart: properties.voyage_start || currentDate,
                
                startCoordinates: coordinates 
            };
        } else {
            // Als we een eerdere datum vinden, updaten we de startgegevens
            if (new Date(currentDate) < new Date(shipMap[log_id].startDate)) {
                 shipMap[log_id].startDate = currentDate;
                 shipMap[log_id].startCoordinates = coordinates;
            }
        }
    });

    // 2. Converteer de Map van unieke schepen naar een array
    // Sorteer op de gevonden vroegste logboekdatum (startDate)
    return Object.values(shipMap).sort((a, b) => new Date(a.startDate) - new Date(b.startDate));
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

    // Allow parent to instruct map to fly to a given [lng, lat]
    flyToLocation(coords, opts = {}) {
      if (!this.map || !coords || coords.length < 2) return;
      const [lng, lat] = coords;
      this.map.flyTo(Object.assign({ center: [lng, lat], zoom: opts.zoom ?? 8, speed: 0.8, essential: true }, opts));
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
