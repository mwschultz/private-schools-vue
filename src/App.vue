<template>
  <div id="app">
    <h1>Private Schools Data</h1>
    <div v-if="loading">Loading...</div>
    <div v-else-if="error">{{ error }}</div>
    <div v-else>
      <div id="map" style="height: 500px; width: 100%;"></div>
    </div>
  </div>
</template>

<script>
import axios from 'axios';

/* global google */

export default {
  data() {
    return {
      data: null,
      loading: true,
      error: null,
      token: null,
      map: null,
      geocoder: null,
    };
  },
  computed: {
    formattedData() {
      return JSON.stringify(this.data, null, 2);
    }
  },
  async created() {
    try {
      await this.fetchToken();
      await this.fetchData();
    } catch (error) {
      this.error = 'Failed to fetch data';
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
        const response = await axios.get("https://private-schools.onrender.com/api/v1/schools/", {
          headers: {
            Authorization: `Bearer ${this.token}`,
          },
        });
        this.data = response.data.data;
        this.loadGoogleMaps();
      } catch (error) {
        this.error = "Failed to fetch data";
        throw error;
      }
    },

    loadGoogleMaps() {
      const script = document.createElement('script');
      script.src = `https://maps.googleapis.com/maps/api/js?key=${process.env.VUE_APP_GOOGLE_MAPS_API_KEY}&callback=initMap`;
      script.async = true;
      script.defer = true;
      window.initMap = this.initMap;
      document.head.appendChild(script);
    },

    initMap() {
      if (typeof google !== 'undefined') {
        this.map = new google.maps.Map(document.getElementById('map'), {
          center: { lat: 35.7796, lng: -78.6382 }, // Center the map to North Carolina
          zoom: 7,
        });
        this.geocoder = new google.maps.Geocoder();
        this.addMarkers();
      } else {
        console.error('Google Maps API is not loaded.');
      }
    },

    addMarkers() {
      this.data.forEach(school => {
        const address = `${school.attributes.street}, ${school.attributes.city}, ${school.attributes.state}`;
        this.geocodeAddress(address, school.attributes);
      });
    },

    geocodeAddress(address, attributes) {
      this.geocoder.geocode({ address: address }, (results, status) => {
        if (status === 'OK') {
          const marker = new google.maps.Marker({
            map: this.map,
            position: results[0].geometry.location,
            title: attributes.name,
          });

          const infoWindow = new google.maps.InfoWindow({
            content: `
              <div>
                <h3>${attributes.name}</h3>
                <p><strong>Headmaster Email:</strong> ${attributes.headmaster}</p>
                <p><strong>Email:</strong> ${attributes.email}</p>
                <p><strong>Address:</strong> ${address}</p>
              </div>
            `,
          });

          marker.addListener('mouseover', () => {
            infoWindow.open(this.map, marker);
          });

          marker.addListener('mouseout', () => {
            infoWindow.close();
          });
        } else {
          console.error('Geocode was not successful for the following reason: ' + status);
        }
      });
    },
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

pre {
  text-align: left;
  background: #f4f4f4;
  padding: 10px;
  border-radius: 5px;
  overflow: auto;
}

#map {
  height: 500px;
  width: 100%;
}
</style>