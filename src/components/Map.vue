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
        geojsonUrl: "/shipslogs_dummy.geojson",
    };
  },

  mounted() {
    const { center, zoom } = this.modelValue

    // instantiate the map using the center and zoom from the modelValue prop
    const map = new mapboxgl.Map({
      container: this.$refs.mapContainer,
      style: "mapbox://styles/saraamelia/cmhvvt5uq000q01s98mrh4gt3",
      center: center,
      zoom,
    });


    map.on('load', this.addGeoJsonLayer);
    const updateLocation = () =>
        this.$emit("update:modelValue", this.getLocation());

    map.on("move", updateLocation);
    map.on("zoom", updateLocation);

    this.map = map;

    // CHANGES BEFORE DUMMY GEOJSON DATA

    /* 
        // function to update the modelValue prop with the map's current location
        const updateLocation = () =>
        this.$emit("update:modelValue", this.getLocation());

        // add event listeners to update the location on map move and zoom
        map.on("move", updateLocation);
        map.on("zoom", updateLocation);

        // assign the map instance to this component's map property
        this.map = map;

    */
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

        const data = this.geojsonUrl; // Using local dummy data path for now

        const transformedData = data;//this.processRawDataToGeoJson(data);

        // 2. Add Source 
        this.map.addSource('ship-log-source', {
          type: 'geojson',
          data: transformedData // GeoJSON FeatureCollection
        });

        // 3. Add layer
        this.map.addLayer({
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
            'circle-stroke-width': 2,
            'circle-stroke-color': '#fff'
          }
        });

        //Interactivtity: mousehovers and popups
        this.addMapInteractivity();

      } catch (error) {
        console.error("Issue loading GEOJson:", error);
      }
    },

    addMapInteractivity() {
        this.map.on('click', 'ship-log-points', (e) => {
            const properties = e.features[0].properties;

            const eventsList = Array.isArray(properties.events) ? properties.events.join(', ') : properties.events;

            const popupHTML = `
              <h3>${properties.ship_name} (${properties.date})</h3>
              <strong>Locatie:</strong> ${e.lngLat.lng.toFixed(2)}, ${e.lngLat.lat.toFixed(2)}<br>
              <strong>Weer:</strong> ${properties.weather}<br>
              <strong>Gebeurtenissen:</strong> ${eventsList || 'Geen bijzondere gebeurtenissen'}
            `;

            new mapboxgl.Popup()
              .setLngLat(e.lngLat)
              .setHTML(popupHTML)
              .addTo(this.map);
        });
        
        // Change cursor at hover
        this.map.on('mouseenter', 'ship-log-points', () => {
            this.map.getCanvas().style.cursor = 'pointer';
        });
        this.map.on('mouseleave', 'ship-log-points', () => {
            this.map.getCanvas().style.cursor = '';
        });
    },

    processRawDataToGeoJson(rawData) {
    const allFeatures = [];
    

    console.log( typeof rawData );

    if (!Array.isArray(rawData)) {
        console.error("Fout: Invoer data is geen array. Kan niet verwerken.");
        return { type: "FeatureCollection", features: [] };
    }

    // Loop door elk schip in de array
    rawData.forEach(shipData => {
        
        // Log om te controleren of het schip is geladen
        console.log(`Verwerking gestart voor schip: ${shipData.ship_name}`);
        
        // Controleer of de entries array bestaat
        if (Array.isArray(shipData.entries)) {
            
            // Loop door elke dagelijkse logboekvermelding
            shipData.entries.forEach(entry => {
                
                // Filter: alleen punten met geldige (niet-null) coördinaten
                if (entry.latitude !== null && entry.longitude !== null && entry.longitude !== undefined) {
                    
                    // Log de geldige coördinaten
                    console.log(`  -> Punt gevonden: ${entry.date}, Lng: ${entry.longitude}, Lat: ${entry.latitude}`);

                    allFeatures.push({
                        "type": "Feature",
                        "geometry": {
                            // BELANGRIJK: GeoJSON/Mapbox gebruikt [Longitude, Latitude]
                            "type": "Point",
                            "coordinates": [entry.longitude, entry.latitude] 
                        },
                        "properties": {
                            // Algemene schip/reis metadata
                            "ship_name": shipData.ship_name,
                            "voyage_id": shipData.log_id,
                            "master": shipData.master,
                            "destination": shipData.destination,
                            
                            // Dagelijkse entry properties
                            "date": entry.date,
                            "weather": entry.weather,
                            "remarks": entry.remarks,
                            
                            // Converteer de 'events' array naar een bruikbare string voor de popup
                            "events": Array.isArray(entry.events) ? entry.events.join('; ') : entry.events
                        }
                    });
                } else {
                    console.log(`  -> Overgeslagen: ${entry.date} heeft ongeldige of missende coördinaten.`);
                }
            });
        } else {
             console.warn(`Waarschuwing: Schip ${shipData.ship_name} mist de 'entries' array of deze is ongeldig.`);
        }
    });

        // Het finale resultaat
        console.log(`Transformatie voltooid. Totaal ${allFeatures.length} punten gegenereerd.`);
        return {
            "type": "FeatureCollection",
            "features": allFeatures
        };
    },
  },
};

</script>

<style>
.map-container {
  width: 100%;
  height: 100%;
}
</style>
