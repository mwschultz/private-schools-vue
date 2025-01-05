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
      <!-- Select counties -->
      <div class="county-selection">
        <h2>Filter By Counties</h2>
        <div v-if="loadingCounties">Loading counties...</div>
        <div v-else-if="countyError">{{ countyError }}</div>
        <div v-else>
          <div class="counties-grid-container">
            <div class="counties-grid">
              <div
                v-for="county in sortedCounties"
                :key="county.id"
                class="county-checkbox-wrapper"
              >
                <label class="county-checkbox">
                  <input
                    type="checkbox"
                    :value="county.id"
                    v-model="selectedCountyIds"
                    @change="handleCountySelection"
                  />
                  <span class="county-name">{{ county.attributes.name }}</span>
                </label>
              </div>
            </div>
          </div>
          <div class="selection-summary" v-if="selectedCountyIds.length > 0">
            Selected: {{ selectedCountyIds.length }} counties
            <button @click="clearSelection" class="clear-button">
              Clear All
            </button>
          </div>
        </div>
      </div>

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
            :class="{ highlighted: highlightedSchoolId === school.id }"
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
      counties: [],
      selectedCountyIds: [],
      loadCounties: true,
      countyError: null,
    };
  },

  async created() {
    try {
      await this.fetchToken();
      await Promise.all([this.fetchCounties(), this.fetchData()]);
    } catch (error) {
      this.error = "Failed to fetch data";
    } finally {
      this.loading = false;
    }
  },

    computed: {
    sortedCounties() {
         console.log('Current counties state:', this.counties);
      return [...this.counties].sort((a, b) => 
        a.attributes.name.localeCompare(b.attributes.name)
      );
    }
  },

  methods: {
    // async initializeData() {
    //   try {
    //     this.loading = true;
    //     this.error = null;
    //     await this.fetchToken();
    //     await this.fetchData();
    //   } catch (error) {
    //     console.error("Initialization error:", error);
    //     this.error = "Failed to load schools data. Please try again.";
    //   } finally {
    //     this.loading = false;
    //   }
    // },

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
        console.error("Token fetch error:", error);
        throw new Error("Authentication failed");
      }
    },

    async fetchCounties() {
      try {
        this.loadingCounties = true;
        const response = await axios.get(
          "https://private-schools.onrender.com/api/v1/counties",
          {
            headers: {
              Authorization: `Bearer ${this.token}`,
            },
          }
        );
        console.log('Counties response:', response.data);
        this.counties = response.data.data;
      } catch (error) {
        this.countyError = "Failed to load counties";
      } finally {
        this.loadingCounties = false;
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
        console.error("Data fetch error:", error);
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
      const schoolCard = document.querySelector(
        `[data-school-id="${markerId}"]`
      );
      if (schoolCard) {
        schoolCard.scrollIntoView({ behavior: "smooth", block: "center" });
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
      immediate: true,
    },
  },
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
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
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
  min-height: 500px; 
  position: relative; 
}

.counties-grid-container {
  max-height: 300px;
  overflow-y: auto;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  background: #f8fafc;
  padding: 10px;
}

.counties-grid {
  display: grid;
  grid-template-columns: repeat(10, 1fr);
  gap: 8px;
  padding: 4px;
}

.county-checkbox-wrapper {
  min-width: 0;
}

.county-checkbox {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 0.9em;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  padding: 4px;
  cursor: pointer;
}

.county-checkbox:hover {
  background-color: #e2e8f0;
  border-radius: 4px;
}

.county-name {
  overflow: hidden;
  text-overflow: ellipsis;
}

.selection-summary {
  margin-top: 10px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px;
  background-color: #e2e8f0;
  border-radius: 4px;
  font-size: 0.9em;
}

.clear-button {
  background-color: #ef4444;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 4px 8px;
  font-size: 0.8em;
  cursor: pointer;
  transition: background-color 0.2s;
}

.clear-button:hover {
  background-color: #dc2626;
}

/* Scrollbar styling */
.counties-grid-container::-webkit-scrollbar {
  width: 8px;
}

.counties-grid-container::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 4px;
}

.counties-grid-container::-webkit-scrollbar-thumb {
  background: #94a3b8;
  border-radius: 4px;
}

.counties-grid-container::-webkit-scrollbar-thumb:hover {
  background: #64748b;
}

/* Responsive adjustments */
@media (max-width: 1200px) {
  .counties-grid {
    grid-template-columns: repeat(8, 1fr);
  }
}

@media (max-width: 768px) {
  .counties-grid {
    grid-template-columns: repeat(5, 1fr);
  }
}

@media (max-width: 480px) {
  .counties-grid {
    grid-template-columns: repeat(3, 1fr);
  }
}
</style>