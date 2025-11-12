<template>
  <div id="app">
    
    <div class="map-container">
      
      <div id="overview-panel">
        <p v-if="shipList.length > 0">Total {{ shipList.length }} ships found.</p>
        <p v-else>Loading map...</p>
      </div>

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
          Click on the points on the map to see logbook details...
        </div>
      </div>
      
      <Map v-model="location" 
           @open-sidebar="openSidebarWithData" 
           @data-loaded="handleDataLoaded"
           ref="mapComponent" />

    </div>

    <div v-show="showWelcomeScreen" id="welcome-screen-overlay"
         :class="{ 'slide-out': welcomeClosing }">
      <h1>⚓ Akira's Historical Logbooks</h1>
      <p>Discover routes and ships of the 18th and 19th century</p>
      
      <button @click="startMapExploration">Start exploring</button>
      
    </div>

  </div>
</template> 

<script>
import Map from "./components/Map.vue";

export default {
  components: { Map },

  data() {
    return {
      // Statusvariabelen op root-niveau
      showWelcomeScreen: true, 
      welcomeClosing: false,
      sidebarOpen: false,
      selectedLog: null,
      
      // Kaart parameters
      location: {
        center: { lng: -10.0, lat: 35.0 },
        zoom: 5,
      },
      
      // Data van de kaart
      totalShips: 0,
      shipList: [] 
    };
  },
  computed: {
    // Formatteert de coördinaten voor de zijbalk
    displayPosition() {
      const log = this.selectedLog;
      
      if (log && typeof log.longitude === 'number' && typeof log.latitude === 'number') {
        return `${log.longitude.toFixed(2)}, ${log.latitude.toFixed(2)}`;
      } 
      return 'Niet geregistreerd';
    }
  },
  methods: {
    // Opent de zijbalk met de geselecteerde logboekvermelding
    openSidebarWithData(logProperties) {
      this.selectedLog = logProperties;
      this.sidebarOpen = true; 
    },
    
    // Ontvangt de lijst met unieke schepen van de Map component
    handleDataLoaded(ships) {
      this.shipList = ships; 
      this.totalShips = ships.length; 
    },
    
    // Verbergt de overlay om de kaart te tonen
    startMapExploration() {
      // Trigger CSS animation, then hide overlay after transition
      this.welcomeClosing = true;
      // match this timeout with the CSS transition duration (600ms)
      setTimeout(() => {
        this.showWelcomeScreen = false;
        this.welcomeClosing = false;
      }, 650);
    }
  }
};
</script>

<style>
/* Zorg dat #app de volledige hoogte heeft voor correcte positionering */
#app {
  height: 100vh;
  width: 100vw;
  position: relative; /* Belangrijk voor absolute kinderen */
}

/* Zorg dat de map-container de volle breedte en hoogte inneemt */
.map-container {
  height: 100%;
  width: 100%;
  position: relative;
}

/* --- CSS VOOR HET WELKOMSTSCHERM (OVERLAY) --- */
#welcome-screen-overlay{
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;

  /* Zorgt dat het BOVEN de kaart ligt */
  z-index: 999; 
  
  /* Styling en centrering */
  /* TRANSPARANTE ACHTERGROND: 70% dekkend perkament */
  background-color: rgba(181, 170, 149, 0.882); 
  color: #151e3c;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  text-align: center;
  gap: 15px; /* Ruimte tussen elementen */

  /* Smooth fade + translate transition */
  transform: translateY(0);
  opacity: 1;
  transition: transform 0.6s ease, opacity 0.6s ease;
}

#welcome-screen-overlay.slide-out {
  /* Schuift het scherm 100% van de viewport-hoogte naar boven */
  transform: translateY(-100vh); 
  opacity: 0;
}
#welcome-screen-overlay h1 {
  font-size: 3em;
  color: #14173c;
  margin-bottom: 0;
}
#welcome-screen-overlay p {
  font-size: 1.2em;
}

#welcome-screen-overlay button {
  padding: 15px 30px;
  font-size: 1.2em;
  background-color: #1b1443; /* Historische, warme kleur */
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  margin-top: 20px;
  transition: background-color 0.3s;
}

#welcome-screen-overlay button:hover {
  background-color: #3a359e;
}
/* Einde Welcome Screen Styles */


/* --- CSS VOOR DE ZIJBALK --- */
#info-sidebar {
  position: absolute;
  top: 0;
  right: 0;
  width: 0; /* Standaard gesloten */
  height: 100%;
  background-color: rgba(198, 211, 245, 0.766);
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
/* Einde Sidebar Styles */

/* --- CSS VOOR OVERVIEW PANEL --- */
#overview-panel {
  position: absolute;
  top: 10px;
  left: 10px;
  z-index: 10;
  padding: 10px;
  background-color: rgba(255, 255, 255, 0.382);
  border-radius: 4px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}
</style>