<template>
  <div id="app">
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
        Click on a ship log point to see details here.
      </div>
    </div>

    <MapView ref="mapRef" v-model="location" @open-sidebar="openSidebarWithData" @ships-loaded="onShipsLoaded" />
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
    // Zorgt voor veilige rendering van de coördinaten
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
    }
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
  color: #666;
}
.badge {
  display: inline-block;
  background: linear-gradient(180deg,#fff,#f0f0f0);
  border: 1px solid #ddd;
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
</style>
