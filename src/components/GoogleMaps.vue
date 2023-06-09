<template>
  <!-- Search Location Form -->
  <div class="container mt-5" style="position: relative; z-index: 1">
    <div class="row d-flex justify-content-center align-items-center">
      <div class="col-md-8">
        <form id="Form">
          <div class="form-group">
            <!-- Display Error Msg -->
            <div class="alert alert-danger" role="alert" v-show="errorMsg">{{ errorMsg }}</div>
            <div class="form-row">
              <div class="col-11">
                <input type="text" class="form-control" v-model="searchInput" @keyup.enter.prevent="searchLocation()" />
              </div>
              <div class="col" style="padding-top: 5px">
                <span id="Search-icon" @click="currentLocation">
                  <i class="fa fa-home"></i>
                </span>
              </div>
            </div>
            <!-- Display Time Zone -->
            <div class="form-row" v-show="timeZone">
              <div class="col-11" style="padding-top: 10px">
                <span class="badge badge-pill badge-info"> Local Time: {{ this.timeDate }}, {{ this.timeZone }}</span>
              </div>
            </div>
          </div>
          <button id="Search-button" type="submit" @click.prevent="searchLocation()" v-show="!spinner">Search</button>
          <div class="spinner-border" role="status" v-show="spinner">
            <span class="sr-only">Loading...</span>
          </div>
        </form>
      </div>
    </div>
  </div>

  <!-- GoogleMap -->
  <div class="container mt-5" id="Map">
    <GoogleMap :api-key="this.$API_KEY" style="width: 100%; height: 700px" :center="center" :zoom="13">
      <MarkerCluster>
        <Marker v-for="(location, i) in locations" :options="{ position: location.location }" :key="i" />
      </MarkerCluster>
    </GoogleMap>
  </div>

  <!-- Location Table -->

  <div id="Table" v-show="locations.length > 0">
    <div class="row d-flex justify-content-center align-items-center">
      <table class="table table-striped">
        <thead>
          <tr>
            <th cope="col"></th>
            <th cope="col">Location</th>
            <th cope="col">Latitude</th>
            <th cope="col">Longitude</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="ele in displayLocation" :key="ele.address">
            <td>
              <input type="checkbox" v-model="selectedLocations" :value="ele.address" />
            </td>
            <td>{{ ele.address }}</td>
            <td>{{ ele.location.lat }}</td>
            <td>{{ ele.location.lng }}</td>
          </tr>
        </tbody>
      </table>
      <!-- Delete Button and Pagination -->
      <div class="container">
        <div class="row">
          <div class="col-sm">
            <button type="button" class="btn btn-danger" @click="deletedSelectedLocations">Delete</button>
          </div>
          <div class="col-sm"></div>
          <div class="col-sm" style="padding-left: 100px">
            <button type="button" class="btn btn-light btn-sm" @click="prevPage" :disabled="currentPage == 1">
              Previous
            </button>
            <span style="padding: 0 5px">{{ currentPage }} / {{ lastPage }}</span>
            <button type="button" class="btn btn-light btn-sm" @click="nextPage" :disabled="currentPage == lastPage">
              Next
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { GoogleMap, Marker, MarkerCluster } from 'vue3-google-map';

import axios from 'axios';
import '../assets/theme.css';

export default {
  components: { GoogleMap, Marker, MarkerCluster },
  data() {
    return {
      searchInput: '',
      errorMsg: '',
      center: { lat: 45.508888, lng: -73.561668 },
      locations: [],
      timeZone: '',
      timeDate: '',
      spinner: false,
      // For Table
      pageSize: 10,
      currentPage: 1,
      lastPage: '',
      selectedLocations: [],
    };
  },
  computed: {
    displayLocation() {
      let start = (this.currentPage - 1) * this.pageSize;
      let end = this.currentPage * this.pageSize;
      return this.locations.slice(start, end);
    },
  },
  methods: {
    currentLocation() {
      this.spinner = true;
      this.errorMsg = '';

      if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition(
          (position) => {
            this.center = {
              lat: position.coords.latitude,
              lng: position.coords.longitude,
            };

            // If Search Location not exist, then push into Locations
            if (!this.locations.some((e) => e.address == 'Current Location')) {
              //console.log('Current Location ' + this.center.lat + ' ' + this.center.lng);
              this.locations.push({ address: 'Current Location', location: this.center });
            }
            this.spinner = false;
            this.getTimeZone();
            this.updatePageInfo();
          },
          // User Denied Geolocation
          (error) => {
            this.errorMsg = 'Error: ' + error.message;
            this.spinner = false;
            // console.log(error.message);
          }
        );
      } else {
        this.spinner = false;
        console.log('Your browser does not support geolocation API ');
      }
    },

    async searchLocation() {
      this.spinner = true;
      this.errorMsg = '';

      try {
        const res = await axios.get('https://maps.googleapis.com/maps/api/geocode/json', {
          params: {
            address: this.searchInput,
            key: this.$API_KEY,
          },
        });

        // console.log('Search Location:' + JSON.stringify(res));

        if (res.data.status == 'OK') {
          this.center = res.data.results[0].geometry.location;
          // If Search Location not exist, then push into Locations
          if (!this.locations.some((e) => e.address == this.searchInput)) {
            // console.log(this.searchInput + ' ' + this.center.lat + ' ' + this.center.lng);
            this.locations.push({ address: this.searchInput, location: this.center });
          }
          this.getTimeZone();
          this.updatePageInfo();
          // console.log(JSON.stringify(this.locations));
        } else {
          //   console.log('Location not found.');
          this.errorMsg = 'Location not found';
        }
        this.spinner = false;
      } catch (error) {
        // console.log(error);
        this.errorMsg = error;
        this.spinner = false;
      }
    },
    // Get Time Zone by latitude and longitude
    async getTimeZone() {
      let timestamp = Date.now();

      try {
        const res = await axios.get(' https://maps.googleapis.com/maps/api/timezone/json', {
          params: {
            location: this.center.lat + ',' + this.center.lng,
            timestamp: timestamp / 1000,
            key: this.$API_KEY,
          },
        });

        if (res.data.status == 'OK') {
          this.timeZone = res.data.timeZoneId;
          // console.log(res.data.timeZoneId);
          let date = new Date(timestamp);
          this.timeDate = date.toLocaleString('en-GB', { timeZone: this.timeZone });
        } else {
          this.errorMsg = 'Location time zone not found.';
        }
      } catch (error) {
        // console.log(error);
        this.errorMsg = error;
      }
    },
    // For Location Table Method
    nextPage() {
      if (this.currentPage * this.pageSize < this.locations.length) this.currentPage++;
    },
    prevPage() {
      if (this.currentPage > 1) this.currentPage--;
    },
    updatePageInfo() {
      this.lastPage = Math.ceil(this.locations.length / this.pageSize);
    },
    deletedSelectedLocations() {
      // console.log(JSON.stringify(this.selectedLocations));
      this.locations = this.locations.filter((location) => !this.selectedLocations.includes(location.address));
      this.selectedLocations = [];
      this.updatePageInfo();
    },
  },
};
</script>
