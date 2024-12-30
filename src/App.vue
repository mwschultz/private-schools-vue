<template>
  <div id="app">
    <h1>Private Schools in North Carolina</h1>
    <div v-if="loading">Loading...</div>
    <div v-else-if="error">{{ error }}</div>
    <div v-else class="main-container">
      <div class="map-container">
        <div id="map"></div>
      </div>
      <div class="schools-list">
        <div class="schools-grid">
          <div
            v-for="school in data"
            :key="school.id"
            class="school-card"
            @mouseover="highlightMarker(school.id)"
            @mouseout="resetMarker(school.id)"
          >
            <h3>{{ school.attributes.name }}</h3>
            <div class="school-details">
              <p><strong>Address:</strong> {{ school.attributes.street }}</p>
              <p>{{ school.attributes.city }}, {{ school.attributes.state }}</p>
              <p><strong>County:</strong> {{ school.attributes.county }}</p>
            </div>
          </div>
        </div>
      </div>
    </div>
    <FooterComponent />
  </div>
</template>

<script>
import axios from "axios";
import FooterComponent from "./components/Footer.vue";

/* global google */

export default {
  components: {
    FooterComponent,
  },
  data() {
    return {
      data: null,
      loading: true,
      error: null,
      token: null,
      map: null,
      geocoder: null,
      markers: new Map(),
    };
  },
  computed: {
    formattedData() {
      return JSON.stringify(this.data, null, 2);
    },
  },
  async created() {
    try {
      await this.fetchToken();
      await this.fetchData();
    } catch (error) {
      this.error = "Failed to fetch data";
    } finally {
      this.loading = false;
    }
  },
  methods: {
    async fetchToken() {
      try {
        const response = await axios.post(
          "https://rails-apis.us.auth0.com/oauth/token",
          {
            client_id: process.env.VUE_APP_NC_PRIVATE_SCHOOLS_ID,
            client_secret: process.env.VUE_APP_NC_PRIVATE_SCHOOLS_CLIENT_SECRET,
            audience: process.env.VUE_APP_NC_PRIVATE_SCHOOLS_CLIENT_AUDIENCE,
            grant_type: process.env.VUE_APP_NC_PRIVATE_SCHOOLS_GRANT_TYPE,
          },
          {
            headers: {
              "content-type": "application/json",
            },
          }
        );
        this.token = response.data.access_token;
      } catch (error) {
        this.error = "Failed to fetch token";
        throw error;
      }
    },

    async fetchData() {
      try {
        const response = await axios.get(
          "https://private-schools.onrender.com/api/v1/schools/",
          {
            headers: {
              Authorization: `Bearer ${this.token}`,
            },
          }
        );

        this.data = response.data.data;
        this.loadGoogleMaps();
      } catch (error) {
        this.error = "Failed to fetch data";
        throw error;
      }
    },

    loadGoogleMaps() {
      if (!document.querySelector(`script[src*="maps.googleapis.com"]`)) {
        const script = document.createElement("script");
        script.src = `https://maps.googleapis.com/maps/api/js?key=${process.env.VUE_APP_GOOGLE_MAPS_API_KEY}&callback=initMap`;
        script.async = true;
        script.defer = true;
        script.setAttribute("loading", "async");
        window.initMap = this.initMap;
        document.head.appendChild(script);
      }
    },

    initMap() {
      if (typeof google !== "undefined") {
        this.map = new google.maps.Map(document.getElementById("map"), {
          center: { lat: 35.7796, lng: -78.6382 }, // Center the map to North Carolina
          zoom: 7,
        });
        this.geocoder = new google.maps.Geocoder();
        this.addMarkers();
      } else {
        console.error("Google Maps API is not loaded.");
      }
    },

    async geocodeAddress(address, attributes, schoolId) {
      this.geocoder.geocode({ address: address }, (results, status) => {
        if (status === "OK") {
          const marker = new google.maps.Marker({
            map: this.map,
            position: results[0].geometry.location,
            title: attributes.name,
            icon: {
              path: google.maps.SymbolPath.CIRCLE,
              scale: 8,
              fillColor: "#2c3e50",
              fillOpacity: 1,
              strokeWeight: 2,
              strokeColor: "#ffffff",
            },
          });

          // Store the marker reference
          this.markers.set(schoolId, marker);

          const infoWindow = new google.maps.InfoWindow({
            content: `
              <div>
                <h3>${attributes.name}</h3>
                <p><strong>Address:</strong> ${address}</p>
              </div>
            `,
          });

          marker.addListener("mouseover", () => {
            infoWindow.open(this.map, marker);
          });

          marker.addListener("mouseout", () => {
            infoWindow.close();
          });
        } else {
          console.error(
            "Geocode was not successful for the following reason: " + status
          );
        }
      });
    },

    addMarkers() {
      this.data.forEach((school) => {
        const address = `${school.attributes.street}, ${school.attributes.city}, ${school.attributes.state}`;
        this.geocodeAddress(address, school.attributes, school.id);
      });
    },

    highlightMarker(schoolId) {
      const marker = this.markers.get(schoolId);
      if (marker) {
        marker.setIcon({
          path: google.maps.SymbolPath.CIRCLE,
          scale: 12, // Larger size
          fillColor: "#e74c3c", // Different color (red)
          fillOpacity: 1,
          strokeWeight: 3,
          strokeColor: "#ffffff",
        });

        // Optionally pan to the marker
        this.map.panTo(marker.getPosition());
      }
    },

    resetMarker(schoolId) {
      const marker = this.markers.get(schoolId);
      if (marker) {
        marker.setIcon({
          path: google.maps.SymbolPath.CIRCLE,
          scale: 8,
          fillColor: "#2c3e50",
          fillOpacity: 1,
          strokeWeight: 2,
          strokeColor: "#ffffff",
        });
      }
    },
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  color: #2c3e50;
  margin: 20px;
}

h1 {
  text-align: center;
  margin-bottom: 30px;
}

.main-container {
  display: flex;
  gap: 20px;
  height: calc(100vh - 200px);
  min-height: 500px;
}

.map-container {
  flex: 1;
  min-width: 0;
}

#map {
  height: 100%;
  width: 100%;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.schools-list {
  flex: 1;
  min-width: 0;
  overflow-y: auto;
  padding-right: 10px;
}

.schools-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 20px;
  padding: 10px;
}

.school-card {
  background: white;
  border-radius: 8px;
  padding: 15px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  text-align: left;
  transition: all 0.2s ease; /* Add smooth transition */
  cursor: pointer; /* Add pointer cursor to indicate interactivity */
}

.school-card:hover {
  background-color: #f5f5f5; /* Light grey background on hover */
  transform: translateY(-2px); /* Slight lift effect */
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.15); /* Enhanced shadow on hover */
}

.school-card h3 {
  margin-top: 0;
  margin-bottom: 10px;
  color: #2c3e50;
}

.school-details p {
  margin: 5px 0;
  font-size: 0.9em;
}

/* Responsive design */
@media (max-width: 768px) {
  .main-container {
    flex-direction: column;
    height: auto;
  }

  .map-container {
    height: 400px;
  }

  .schools-list {
    height: 500px;
  }
}

/* Scrollbar styling */
.schools-list::-webkit-scrollbar {
  width: 8px;
}

.schools-list::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 4px;
}

.schools-list::-webkit-scrollbar-thumb {
  background: #888;
  border-radius: 4px;
}

.schools-list::-webkit-scrollbar-thumb:hover {
  background: #555;
}
</style>