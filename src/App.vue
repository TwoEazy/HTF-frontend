<template>
  <div id="app">
    
    <div class="map-container">
      
      <div id="info-sidebar" :class="{ 'open': sidebarOpen }">
        <button class="close-btn" @click="sidebarOpen = false">X</button>
  
        <div v-if="selectedLog" class="content details">
          <div class="ship-header">
          <div class="ship-icon">🚢</div>
          <div>
            <h2 style="margin:0">{{ selectedLog.ship_name || 'Unknown vessel' }}</h2>
            <div class="meta">Master: {{ selectedLog.master || '—' }} • Voyage: {{ selectedLog.voyage_start || '—' }} → {{ selectedLog.voyage_end || '—' }}</div>
          </div>
        </div>

          <h3 style="margin:6px 0 8px 0">Logbook: {{ formatDate(selectedLog.date) }}</h3>
  
    <p><strong>Position:</strong> {{ displayPosition }}</p>
  
          <p><strong>Weather:</strong> <span class="badge">{{ selectedLog.weather || '—' }}</span></p>

          <p><strong>Remarks:</strong></p>
        <p>{{ selectedLog.remarks || '—' }}</p>

          <div>
          <strong>Events</strong>
          <ul class="events">
            <li v-for="(ev, idx) in (selectedLog.events || [])" :key="idx">{{ ev }}</li>
            <li v-if="!(selectedLog.events && selectedLog.events.length)">No events recorded</li>
          </ul>
        </div>

        <details style="margin-top:8px;font-size:13px;color:#666">
          <summary>Raw data</summary>
          <pre style="white-space:pre-wrap;word-break:break-word">{{ JSON.stringify(selectedLog, null, 2) }}</pre>
          </details>
      </div>
        <div v-else class="content">
          Click on the points on the map to see logbook details...
        </div>
      </div>
      

      <MapView ref="mapRef" v-model="location" @open-sidebar="openSidebarWithData" @data-loaded="handleDataLoaded" @ships-loaded="onShipsLoaded" />

    <!-- Left menu for ships -->
    <div id="left-menu" :class="{ open: menuOpen }" @mouseenter="menuOpen = true" @mouseleave="menuOpen = false">
      <button class="menu-toggle" @click="menuOpen = !menuOpen">☰ Ships</button>
      <div class="menu-list">
        <div v-if="ships && ships.length">
          <div v-for="(s, idx) in ships" :key="s.log_id || s.ship_name || idx" class="ship-item">
            <button class="ship-btn" @click="goToShip(s)">
              <div class="ship-name">{{ s.ship_name }}</div>
              <div class="ship-meta">{{ s.startDate ? (new Date(s.startDate)).toLocaleDateString() : '' }}</div>
            </button>
          </div>
        </div>
        <div v-else class="empty">No ships loaded</div>
      </div>
    </div>
    </div>

    <div v-show="showWelcomeScreen" id="welcome-screen-overlay"
         :class="{ 'slide-out': welcomeClosing }"
         @transitionend="onWelcomeTransitionEnd">
      <h1>⚓ Akira's Historical Logbooks</h1>
      <p>Discover routes and ships of the 18th and 19th century</p>
      
      <button @click="startMapExploration">Start exploring</button>
      
    </div>

    
  </div>
</template>

<script>
import MapView from "./components/Map.vue";

export default {
  components: { MapView },

  //Initial location => change for HTF!!!!
  data() {
    return {
      // Map location object kept small for v-model binding
      // Statusvariabelen op root-niveau
      showWelcomeScreen: true,
      welcomeClosing: false,
      // Kaart parameters
      location: {
        center: { lng: -10.0, lat: 35.0 },
        zoom: 5
      },
      // Sidebar state and selected log are top-level for clarity
      sidebarOpen: false,
      selectedLog: null,
      // ship menu
      ships: [],
      menuOpen: true
    };
  },
  computed: {
    // Formatteert de coördinaten voor de zijbalk
    displayPosition() {
      return this.formatLatLng(this.selectedLog);
    }
  },
  methods: {
  // NIEUWE METHODE: Wordt aangeroepen door MapComponent
  openSidebarWithData(logProperties) {
      // Ensure numeric coords if present
      if (logProperties) {
        const lng = Number(logProperties.longitude ?? logProperties.lng ?? logProperties.lon ?? logProperties.long);
        const lat = Number(logProperties.latitude ?? logProperties.lat ?? logProperties.latitud ?? logProperties.latitude);
        if (Number.isFinite(lng)) logProperties.longitude = lng;
        if (Number.isFinite(lat)) logProperties.latitude = lat;
      }

      // Normalize events: API sometimes returns a JSON string or other formats
      if (logProperties) {
        const ev = logProperties.events;
        if (typeof ev === 'string') {
          try {
            const parsed = JSON.parse(ev);
            logProperties.events = Array.isArray(parsed) ? parsed : [parsed];
          } catch {
            // not JSON, try to split by commas as a fallback
            logProperties.events = ev.length ? ev.split(/,\s*/) : [];
          }
        } else if (Array.isArray(ev)) {
          // already fine
        } else if (ev == null) {
          logProperties.events = [];
        } else {
          // wrap single non-string non-array event
          logProperties.events = [ev];
        }
      }

      this.selectedLog = logProperties;
      this.sidebarOpen = true; // Open the sidebar
    },

    // Called when Map.vue emits ships-loaded
    onShipsLoaded(shipsList) {
      // keep a simple, stable list
      this.ships = shipsList || [];
    },

    // Called when user clicks a ship in the left menu
    goToShip(ship) {
      if (!ship || !ship.start) return;
      // Close menu and open sidebar for context
      this.menuOpen = false;
      this.selectedLog = { ship_name: ship.ship_name, date: ship.startDate, longitude: ship.start[0], latitude: ship.start[1] };
      this.sidebarOpen = true;
      // Call child component flyTo via ref
      if (this.$refs.mapRef && this.$refs.mapRef.flyToLocation) {
        this.$refs.mapRef.flyToLocation(ship.start, { zoom: 6 });
      }
    },

    formatDate(dateStr) {
      if (!dateStr) return 'Unknown';
      const d = new Date(dateStr);
      if (isNaN(d)) return dateStr;
      return d.toLocaleDateString(undefined, { day: '2-digit', month: 'short', year: 'numeric' });
    },

    formatLatLng(log) {
      if (!log) return 'Unknown';
      const lat = Number(log.latitude ?? log.lat);
      const lng = Number(log.longitude ?? log.lng ?? log.lon ?? log.long);
      if (!Number.isFinite(lat) || !Number.isFinite(lng)) return 'Niet geregistreerd';
      const latDir = lat >= 0 ? 'N' : 'S';
      const lngDir = lng >= 0 ? 'E' : 'W';
      return `${Math.abs(lat).toFixed(2)}° ${latDir}, ${Math.abs(lng).toFixed(2)}° ${lngDir}`;
    },
    startMapExploration() {
      // trigger CSS animation; overlay will be hidden in onWelcomeTransitionEnd
      this.welcomeClosing = true;
      // fallback in case transitionend doesn't fire (safety)
      this._welcomeTimeout = setTimeout(() => {
        if (this.welcomeClosing) {
          this.showWelcomeScreen = false;
          this.welcomeClosing = false;
        }
      }, 700); // slightly longer than CSS duration (600ms)
    },

    onWelcomeTransitionEnd(e) {
      if (!this.welcomeClosing) return;
      // ensure we only handle when the overlay is finishing its transition
      // (multiple properties may fire; just guard with the flag)
      clearTimeout(this._welcomeTimeout);
      this.showWelcomeScreen = false;
      this.welcomeClosing = false;
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
  background-color: rgba(149, 168, 181, 0.882); 
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
  background-color: rgba(88, 91, 99, 0.918);
  box-shadow: -2px 0 5px rgba(0, 0, 0, 0.5);
  overflow-x: hidden;
  transition: 0.3s;
  z-index: 2; /* Zorg dat deze boven de kaart ligt */
  color: #f0eeee;
}

#info-sidebar.open {
  width: 350px; /* Breedte als de zijbalk open is */
  padding: 20px;
}

#info-sidebar .content {
  padding: 0 10px;
}

/* Sidebar visual improvements */
.ship-header {
  display: flex;
  gap: 12px;
  align-items: center;
  margin-bottom: 8px;
}
.ship-icon {
  font-size: 36px;
}
.meta {
  font-size: 13px;
  color: #bcb9b9;
}
.badge {
  display: inline-block;
  background: linear-gradient(180deg,#7672a2,#1d144f);
  border: 1px solid #202354;
  padding: 4px 8px;
  border-radius: 12px;
  font-size: 12px;
  margin-left: 8px;
}
.events {
  margin: 6px 0 12px 0;
  padding-left: 18px;
}
.details p {
  margin: 8px 0;
  line-height: 1.3;
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

/* Left ship menu */
#left-menu {
  position: absolute;
  left: 12px;
  top: 12px;
  width: 220px;
  max-height: calc(100% - 24px);
  background: rgba(255,255,255,0.95);
  border-radius: 6px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.2);
  overflow: hidden;
  z-index: 3;
  transition: transform 0.18s ease, opacity 0.18s ease;
}
#left-menu .menu-toggle {
  width: 100%;
  text-align: left;
  padding: 8px 12px;
  border: none;
  background: #1f2937;
  color: #fff;
  cursor: pointer;
}
#left-menu .menu-list { padding: 8px; max-height: 60vh; overflow-y: auto; }
.ship-item { margin-bottom: 6px; }
.ship-btn { width: 100%; display:flex; justify-content:space-between; align-items:center; padding:8px; border-radius:4px; border:1px solid #eee; background:#fff; cursor:pointer; }
.ship-btn:hover { background:#f6f6f6 }
.ship-name { font-weight:600 }
.ship-meta { font-size:12px; color:#666 }
.empty { padding:8px; color:#666 }
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
