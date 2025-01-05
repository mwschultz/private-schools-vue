# App.vue
<template>
  <div id="app">
    <h1>Private Schools in North Carolina</h1>
    <div v-if="loading" class="loading-state">
      <div class="loading-spinner"></div>
      Loading...
    </div>
    <div v-else-if="error" class="error-state">
      {{ error }}
      <button @click="retryFetch" class="retry-button">Retry</button>
    </div>
    <div v-else class="main-container">
      <div class="map-container">
        <LeafletMap
          :markers="transformedMarkers"
          :highlightedMarkerId="highlightedSchoolId"
          v-if="transformedMarkers.length > 0"
          @marker-click="handleMarkerClick"
        />
        <div v-else class="loading">Loading locations...</div>
      </div>
      <div class="schools-list">
        <div class="schools-grid">
          <div
            v-for="school in apiData"
            :key="school.id"
            class="school-card"
            :class="{ 'highlighted': highlightedSchoolId === school.id }"
            @mouseover="highlightMarker(school.id)"
            @mouseout="resetMarker"
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
  </div>
  <FooterComponent />
</template>

<script>
import axios from "axios";
import LeafletMap from "./components/LeafletMap.vue";
import FooterComponent from "./components/Footer.vue";

export default {
  components: {
    LeafletMap,
    FooterComponent,
  },
  
  data() {
    return {
      apiData: null,
      loading: true,
      error: null,
      token: null,
      transformedMarkers: [],
      highlightedSchoolId: null,
    };
  },

  async created() {
    await this.initializeData();
  },

  methods: {
    async initializeData() {
      try {
        this.loading = true;
        this.error = null;
        await this.fetchToken();
        await this.fetchData();
      } catch (error) {
        console.error('Initialization error:', error);
        this.error = "Failed to load schools data. Please try again.";
      } finally {
        this.loading = false;
      }
    },

    transformApiData(schools) {
      if (!schools) return [];

      return schools.map((school) => ({
        id: school.id,
        position: [school.attributes.latitude, school.attributes.longitude],
        popup: `
          <strong>${school.attributes.name}</strong><br>
          ${school.attributes.street}<br>
          ${school.attributes.city}, ${school.attributes.state}<br>
          ${school.attributes.county} County
        `,
      }));
    },

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
        console.error('Token fetch error:', error);
        throw new Error("Authentication failed");
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
        this.apiData = response.data.data;
      } catch (error) {
        console.error('Data fetch error:', error);
        throw new Error("Failed to fetch schools data");
      }
    },

    highlightMarker(schoolId) {
      this.highlightedSchoolId = schoolId;
    },

    resetMarker() {
      this.highlightedSchoolId = null;
    },

    handleMarkerClick(markerId) {
      // Scroll the corresponding school card into view
      const schoolCard = document.querySelector(`[data-school-id="${markerId}"]`);
      if (schoolCard) {
        schoolCard.scrollIntoView({ behavior: 'smooth', block: 'center' });
      }
    },

    retryFetch() {
      this.initializeData();
    },
  },

  watch: {
    apiData: {
      handler(newData) {
        this.transformedMarkers = this.transformApiData(newData);
      },
      immediate: true
    }
  }
};
</script>

<style>


.loading-state {
  text-align: center;
  padding: 2rem;
}

.loading-spinner {
  border: 3px solid #f3f3f3;
  border-top: 3px solid #3498db;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  animation: spin 1s linear infinite;
  margin: 0 auto 1rem;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.error-state {
  text-align: center;
  color: #e74c3c;
  padding: 2rem;
}

.retry-button {
  margin-top: 1rem;
  padding: 0.5rem 1rem;
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.retry-button:hover {
  background-color: #2980b9;
}

.school-card.highlighted {
  background-color: #e3f2fd;
  border-left: 4px solid #2196f3;
}

/* Ensure smooth transitions */
.school-card {
  border-left: 4px solid transparent;
  transition: all 0.3s ease;
}

.map-container {
  flex: 1;
  min-width: 0;
  min-height: 500px; /* Add this */
  position: relative; /* Add this */
}
</style>