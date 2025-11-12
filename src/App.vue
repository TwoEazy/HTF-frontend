<template>
  <div id="app">
    <div id="info-sidebar" :class="{ 'open': sidebarOpen }">
      <button class="close-btn" @click="sidebarOpen = false">X</button>
      
      <div v-if="selectedLog" class="content">
        <h2>🚢 {{ selectedLog.ship_name }}</h2>
        <h3>Logbook: {{ selectedLog.date }}</h3>
        
        <p><strong>Position:</strong> {{ displayPosition }}</p>
        
        <p><strong>Weather:</strong> {{ selectedLog.weather }}</p>
        <p><strong>Remarkts:</strong> {{ selectedLog.remarks }}</p>
        <p><strong>Events:</strong> {{ selectedLog.events }}</p>
        
        </div>
      <div v-else class="content">
        Click on a ship log point to see details here.
      </div>
    </div>
    
    <Map v-model="location" @open-sidebar="openSidebarWithData" />
  </div>
</template> 

<script>
import Map from "./components/Map.vue";

export default {
  components: { Map },

  //Initial location => change for HTF!!!! 
  data() {
    return {
      location: {
        center: { lng: -10.0, lat: 35.0 },
        zoom: 5,
        sidebarOpen: false,
        selectedLog: null,
      },
    };
  },
  computed: {
    // Zorgt voor veilige rendering van de coördinaten
    displayPosition() {
      const log = this.selectedLog;
      
      // Controleert of coördinaten bestaan en numeriek zijn
      if (log && typeof log.longitude === 'number' && typeof log.latitude === 'number') {
        // Formatteert de coördinaten
        return `${log.longitude.toFixed(2)}, ${log.latitude.toFixed(2)}`;
      } 
      // Fallback
      return 'Niet geregistreerd';
    }
  },
  methods: {
    // NIEUWE METHODE: Wordt aangeroepen door MapComponent
    openSidebarWithData(logProperties) {
      // Zorg ervoor dat Longitude en Latitude numeriek zijn (Mapbox geeft ze niet altijd mee in properties)
      //logProperties.longitude = logProperties.coordinates ? logProperties.coordinates[0] : null;
      //logProperties.latitude = logProperties.coordinates ? logProperties.coordinates[1] : null;

      this.selectedLog = logProperties;
      this.sidebarOpen = true; // Open de zijbalk
    },
  }
};
</script>

<style>
#sidebar {
  background-color: rgb(35 55 75 / 90%);
  color: #fff;
  padding: 6px 12px;
  font-family: monospace;
  z-index: 1;
  position: absolute;
  top: 0;
  left: 0;
  margin: 12px;
  border-radius: 4px;
}

/* --- CSS VOOR DE ZIJBALK --- */
#info-sidebar {
  position: absolute;
  top: 0;
  right: 0;
  width: 0; /* Standaard gesloten */
  height: 100%;
  background-color: rgba(255, 255, 255, 0.95);
  box-shadow: -2px 0 5px rgba(0, 0, 0, 0.5);
  overflow-x: hidden;
  transition: 0.3s;
  z-index: 2; /* Zorg dat deze boven de kaart ligt */
  color: #333;
}

#info-sidebar.open {
  width: 350px; /* Breedte als de zijbalk open is */
  padding: 20px;
}

#info-sidebar .content {
  padding: 0 10px;
}

.close-btn {
  position: absolute;
  top: 10px;
  right: 15px;
  font-size: 20px;
  background: none;
  border: none;
  cursor: pointer;
}
</style>