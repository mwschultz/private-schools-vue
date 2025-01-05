// LeafletMap.vue
<template>
  <div class="map-container">
    <div ref="mapContainer" style="height: 100%; width: 100%;"></div>
  </div>
</template>

<script>
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

import icon from 'leaflet/dist/images/marker-icon.png'
import iconShadow from 'leaflet/dist/images/marker-shadow.png'

export default {
  name: 'LeafletMap',
  
  props: {
    markers: {
      type: Array,
      required: true
    },
    highlightedMarkerId: {
      type: String,
      default: null
    },
    center: {
      type: Array,
      default: () => [35.5175, -80.8304]
    },
    zoom: {
      type: Number,
      default: 7
    }
  },

  data() {
    return {
      map: null,
      markerLayer: null,
      markersMap: new Map(),
      isInitialized: false
    }
  },

  methods: {
    initializeMap() {
      if (this.isInitialized) return;

      // Clear any existing map instance
      if (this.map) {
        this.map.remove();
        this.map = null;
      }

      // Set up default icon
      const DefaultIcon = L.icon({
        iconUrl: icon,
        shadowUrl: iconShadow,
        iconSize: [25, 41],
        iconAnchor: [12, 41]
      });
      L.Marker.prototype.options.icon = DefaultIcon;

      // Initialize map with specific options to handle zoom/scroll issues
      this.map = L.map(this.$refs.mapContainer, {
        zoomAnimation: false, // Disable zoom animation
        fadeAnimation: false, // Disable fade animation
        markerZoomAnimation: false, // Disable marker zoom animation
        zoomSnap: 0.5, // Smoother zooming
        wheelDebounceTime: 150 // Debounce wheel events
      }).setView(this.center, this.zoom);

      // Add tile layer
      L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '© OpenStreetMap contributors',
        maxZoom: 19
      }).addTo(this.map);

      // Initialize marker layer
      this.markerLayer = L.featureGroup().addTo(this.map);

      this.isInitialized = true;

      // Add error handler for zoom events
      this.map.on('zoomanim', (e) => {
        if (!this.map || !this.map._latLngToNewLayerPoint) {
          e.stop();
        }
      });
    },

    updateMarkers() {
      if (!this.map || !this.markerLayer) return;

      try {
        // Clear existing markers
        this.markerLayer.clearLayers();
        this.markersMap.clear();

        // Add new markers
        this.markers.forEach(markerData => {
          if (!markerData.position || !Array.isArray(markerData.position)) return;
          
          const marker = L.marker(markerData.position)
            .bindPopup(markerData.popup)
            .on('click', () => {
              this.$emit('marker-click', markerData.id);
            });
          
          this.markersMap.set(markerData.id, marker);
          marker.addTo(this.markerLayer);
        });

        // Fit bounds if there are markers
        if (this.markers.length > 0 && this.markerLayer.getBounds()) {
          this.map.fitBounds(this.markerLayer.getBounds(), {
            padding: [50, 50],
            maxZoom: 13,
            animate: false // Disable animation for bounds fitting
          });
        }
      } catch (error) {
        console.error('Error updating markers:', error);
      }
    },


  },

  watch: {
    markers: {
      handler() {
        if (this.map && this.isInitialized) {
          this.$nextTick(() => {
            this.updateMarkers();
          });
        }
      },
      deep: true
    }
  },

  mounted() {
    this.$nextTick(() => {
      this.initializeMap();
      if (this.markers.length > 0) {
        this.updateMarkers();
      }
    });
  },

  beforeUnmount() {
    if (this.map) {
      this.map.off();
      this.map.remove();
      this.map = null;
    }
    this.isInitialized = false;
  }
}
</script>

<style scoped>
.map-container {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  height: 100%;
  width: 100%;
}
</style>