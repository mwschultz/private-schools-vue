<template>
  <div id="app">
    <h1>Private Schools Data</h1>
    <div v-if="loading">Loading...</div>
    <div v-else-if="error">{{ error }}</div>
    <pre v-else>{{ formattedData }}</pre>
  </div>
</template>
<script>
import axios from "axios";

export default {
  data() {
    return {
      data: null,
      loading: true,
      error: null,
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
        const response = await axios.get("https://private-schools.onrender.com/api/v1/counties/", {
          headers: {
            Authorization: `Bearer ${this.token}`,
          },
        });
        this.data = response.data;
      } catch (error) {
        this.error = "Failed to fetch data";
        throw error;
      }
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
</style>