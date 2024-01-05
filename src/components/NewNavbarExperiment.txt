<script>
import axios from "axios";

export default {
  data() {
    return {
      searchQuery: "",
      isAdvancedSearchOpen: false,
      isSmallScreen: window.innerWidth < 760,
      filteredApartments: [],
      noApartmentsFound: false,
      checkinDate: null,
      checkoutDate: null,
      searchText: '',                       //indirizzo salvato dall'utente
      results: [],                          //risultati indirizzi
      apartments: [],
      nearbyApartments: [],
      selectedAddress: {                    //array di oggetti che contiene i dati dell'indirizzo cliccato
        address: {},                        //vengono salvati i dati allo stesso modo di come sono salvati nell'API di TomTom
        position: {
          lat: undefined,
          lon: undefined
        },
      },
    };
  },
  computed: {
    showNoApartmentsMessage() {
      return this.noApartmentsFound && !this.isAdvancedSearchOpen;
    },
  },
  methods: {
    getAddress() {                          //chiamata axios per trovare indirizzi, latitudine e longitudine
      axios.get(`https://api.tomtom.com/search/2/geocode/${this.searchText}.json?key=G3UqwADY39DYhuxHmuH49Pv68jOXjJTW`)
        .then((res) => {
          this.results = res.data.results;                                  //tutta la lista di indirizzi
        });
    },
    getApartmentsWithinRadius() {
      if (this.selectedAddress.position.lat === undefined || this.selectedAddress.position.lon === undefined)
        return

      axios.get('http://127.0.0.1:8000/api/apartments')
        .then((res) => {
          res.data.forEach((apartment) => {
            const distance = this.haversineDistance(this.selectedAddress.position.lat, this.selectedAddress.position.lon, apartment.latitude, apartment.longitude);
            if (distance < 20) {
              const nearbyApartmentData = {
                id: apartment.id,
                address: apartment.address,
              }
              this.nearbyApartments.push(nearbyApartmentData);
              console.log(apartment);
            }
          })
        })
        .catch((error) => {
          console.error('Errore durante la chiamata API:', error);
        })
    },
    selectAddress(result) {
      if (result) {
        this.searchText = result.address.freeformAddress;                     //il searchText viene assegnato al valore dell'indirizzo cliccato
        this.selectedAddress = result;                                        //vengono salvati nell'array del "data() return{}" i dettagli dell'indirizzo
      } else {
        this.searchText = this.results[0].address.freeformAddress;                     //il searchText viene assegnato al valore dell'indirizzo cliccato
        this.selectedAddress = this.results[0];
      }

      this.getApartmentsWithinRadius();
    },
    haversineDistance(lat1, lon1, lat2, lon2) {
      // Converte le latitudini e longitudini da gradi a radianti
      const toRadians = (angle) => (angle * Math.PI) / 180;
      const R = 6371; // Raggio medio della Terra in km
      const dLat = toRadians(lat2 - lat1);
      const dLon = toRadians(lon2 - lon1);
      const a =
        Math.sin(dLat / 2) * Math.sin(dLat / 2) +
        Math.cos(toRadians(lat1)) * Math.cos(toRadians(lat2)) * Math.sin(dLon / 2) * Math.sin(dLon / 2);
      const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
      // Calcola la distanza
      const distance = R * c;
      return distance;
    },
    getApartments() {
      axios.get('http://127.0.0.1:8000/api/apartments')
        .then((res) => {
          this.apartments = res.data;
        })
        .catch((error) => {
          console.error('Errore durante la richiesta API:', error);
        });
    },
    toggleAdvancedSearch() {
      this.isAdvancedSearchOpen = !this.isAdvancedSearchOpen;
    },
    handleResize() {
      this.isSmallScreen = window.innerWidth < 760;
    },
    applyFilters() {
      // Make an Axios call to fetch filtered apartments based on the searchQuery
      axios
        .get("http://127.0.0.1:8000/api/apartments", {
          params: { search: this.searchQuery },
        })
        .then((response) => {
          this.filteredApartments = response.data;
          this.noApartmentsFound = false; // Reset the flag when apartments are found
        })
        .catch((error) => {
          console.error("Error fetching filtered apartments:", error);
          this.filteredApartments = []; // Set filteredApartments to an empty array on error
          this.noApartmentsFound = true; // Set the flag when no apartments are found
        });
    },
    applyFiltersOnEnter() {
      // Trigger applyFilters when the "Enter" key is pressed inside the search bar
      this.applyFilters();
    },
    closeAdvancedSearch() {
      // Close the advanced search when clicking outside the search bar or logo
      this.isAdvancedSearchOpen = false;
    },
    handleSearchBarClick(event) {
      // Stop the propagation of the click event to prevent closing the advanced search
      event.stopPropagation();
    },
    handleApartmentClick(apartmentId) {
      // Close the advanced search
      this.isAdvancedSearchOpen = false;

      // Navigate to the individual apartment page
      this.$router.push(`/apartments/${apartmentId}`);
    },
  },
  mounted() {
    this.getApartments();
    window.addEventListener("resize", this.handleResize);
    window.addEventListener("click", this.closeAdvancedSearch);
  },
  beforeDestroy() {
    window.removeEventListener("resize", this.handleResize);
    window.removeEventListener("click", this.closeAdvancedSearch);
  },
};
</script>

<template>
  <div>
    <nav class="navbar navbar-expand-lg navbar-light">
      <div class="container d-flex justify-content-between align-items-center">
        <!-- Left side - LOGO as a clickable link to the home page -->
        <router-link to="/" class="navbar-brand">
          <img v-if="isSmallScreen" src="\src\assets\Risorsa 7port creatt.png" alt="Small Logo" class="small-logo" />
          <img v-else src="\src\assets\Risorsa 4port creatt.png" alt="Large Logo" class="large-logo" />
        </router-link>
        <!-- Burger menu button for smaller screens -->
        <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarNav"
          aria-controls="navbarNav" aria-expanded="false" aria-label="Toggle navigation">
          <span class="navbar-toggler-icon"></span>
        </button>

        <!-- Centered search bar -->
        <div class="d-flex">
          <div class="search-bar-container" @click="handleSearchBarClick">
            <input v-model="searchQuery" @click="toggleAdvancedSearch" @keydown.enter.prevent="applyFiltersOnEnter"
              type="text" class="form-control" placeholder="Dove si va..." />
            <!-- Advanced Search Dropdown -->
            <div v-if="isAdvancedSearchOpen" class="advanced-search-container position-absolute">
              <div class="advanced-search bg-light border rounded p-2 gap-5 d-flex">
                <!-- First Column -->
                <div class="column">
                  <!-- Free range slider for distance -->
                  <div class="form-group p-3">
                    <label>Distanza (km):</label>
                    <input type="range" v-model="distance" min="0" max="100" step="1" class="form-control-range" />
                    <span>{{ distance }} km</span>
                  </div>

                  <!-- Number of beds -->
                  <div class="form-group p-3">
                    <label>Numero Letti:</label>
                    <input type="number" v-model="beds" min="1" class="form-control" />
                  </div>

                  <!-- Number of bathrooms -->
                  <div class="form-group p-3">
                    <label>Numero Bagni:</label>
                    <input type="number" v-model="bathrooms" min="1" class="form-control" />
                  </div>

                  <!-- Number of persons -->
                  <div class="form-group p-3">
                    <label>Numero Persone:</label>
                    <input type="number" v-model="persons" min="1" class="form-control" />
                  </div>
                </div>

                <!-- Second Column -->
                <div class="column">
                  <!-- Checkboxes for services -->
                  <div class="form-group p-3">
                    <label>Servizi:</label>
                    <div v-for="(service, index) in services" :key="index" class="form-check">
                      <input type="checkbox" v-model="selectedServices" :value="service.id" class="form-check-input" />
                      <label class="form-check-label">{{ service.name }}</label>
                    </div>
                  </div>

                  <!-- Calendars -->
                  <div class="form-group p-3">
                    <label>Data Check-in:</label>
                    <input v-model="checkinDate" type="date" class="form-control" />
                  </div>
                  <div class="form-group p-3">
                    <label>Data Check-out:</label>
                    <input v-model="checkoutDate" type="date" class="form-control" />
                  </div>
                </div>

                <!-- Button to apply filters -->
                <div class="button-container">
                  <button @click="applyFilters" class="btn btn-apply-filters">
                    Applica i Filtri
                  </button>
                </div>

                <!-- Display the filtered apartments -->
                <!-- Display the filtered apartments with images -->
                <div v-if="filteredApartments.length" class="search-results">
                  <ul>
                    <div v-for="apartment in filteredApartments" :key="apartment.id"
                      @click="handleApartmentClick(apartment.id)" class="apartment-link-container">
                      <router-link :to="'/apartments/' + apartment.id" tag="div" class="apartment-link">
                        <li>
                          <img :src="apartment.img" alt="Apartment Image" class="apartment-img" />
                          {{ apartment.name }} - {{ apartment.price }}
                        </li>
                      </router-link>
                    </div>
                  </ul>
                  <p class="mini-p" v-if="showNoApartmentsMessage">
                    * Nessun appartamento corrisponde alla ricerca
                  </p>
                </div>
              </div>
            </div>
          </div>
          <div class="container d-flex">

            <div class="d-flex search-bar-container" @click="handleSearchBarClick">

              <input class="form-control" type="text" v-model="searchText" @click="getAddress"
                @keydown.enter="selectAddress()" placeholder="Inserisci un indirizzo" />
              <!-- la ricerca avviene alla pressione di ogni tasto -->
              <button @click="getAddress" type="submit" class="btn btn-primary mx-2">Cerca</button>
              <div class="advanced-search-container position-absolute bg-light border rounded p-2 gap-5 d-flex">
                <div v-if="results.length > 0">
                  <h4>Lista indirizzi più pertinenti:</h4>
                  <div class="inner-container bg-white rounded-3 p-3">
                    <ul>
                      <li class="address-list" v-for="(result, index) in results" :key="index"
                        @click="selectAddress(result)">
                        <!-- al click del mouse, la barra di ricerca viene completata con l'indirizzo selezionato -->
                        {{ result.address.freeformAddress }} <!-- "freeformAddress" è una chiave dell'API TomTom -->
                      </li>
                    </ul>
                  </div>

                  <div>
                    <h4>Lista di proprietà più vicine:</h4>
                    <div class="inner-container bg-white rounded-3 p-3" v-for="nearbyApartment in nearbyApartments">
                      <a :href="/apartments/ + nearbyApartment.id">
                        <h4>{{ nearbyApartment.address }}</h4>
                      </a>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- Right side links for larger screens -->
        <div class="navbar-nav d-none d-lg-flex">
          <router-link to="/" class="nav-link" :class="{ 'font-weight-bold': $route.path === '/' }">
            Home
          </router-link>
          <router-link to="/apartments" class="nav-link" :class="{ 'font-weight-bold': $route.path === '/apartments' }">
            Appartamenti
          </router-link>
          <!-- <router-link
            to="/about"
            class="nav-link"
            :class="{ 'font-weight-bold': $route.path === '/about' }"
          >
            Su di noi
          </router-link> -->
          <router-link to="/login" class="nav-link" :class="{ 'font-weight-bold': $route.path === '/login' }">
            Login
          </router-link>
          <!-- Burger menu for smaller screens -->
          <div class="collapse navbar-collapse" id="navbarNav">
            <div class="navbar-nav d-lg-none">
              <router-link to="/" class="nav-link" :class="{ 'font-weight-bold': $route.path === '/' }">
                Home
              </router-link>
              <router-link to="/apartments" class="nav-link"
                :class="{ 'font-weight-bold': $route.path === '/apartments' }">
                Appartamenti
              </router-link>
              <router-link to="/about" class="nav-link" :class="{ 'font-weight-bold': $route.path === '/about' }">
                Su di noi
              </router-link>
              <router-link to="/login" class="nav-link" :class="{ 'font-weight-bold': $route.path === '/login' }">
                Login
              </router-link>
            </div>
          </div>
        </div>
      </div>
    </nav>
  </div>
</template>

<style lang="scss" scoped>
@media (max-width: 994px) {
  .small-logo {
    display: block;
  }

  .large-logo {
    display: none;
  }

  .navbar-nav.d-lg-flex {
    display: none;
  }
}

@media (min-width: 995px) {
  .small-logo {
    display: none;
  }

  .large-logo {
    display: block;
  }

  .navbar-nav.d-lg-none {
    display: none;
  }
}

@media (max-width: 760px) {
  .small-logo {
    display: block;
  }

  .large-logo {
    display: none;
  }
}

@media (min-width: 761px) {
  .small-logo {
    display: none;
  }

  .large-logo {
    display: block;
  }
}

@media (max-width: 460px) {
  .navbar-brand {
    display: none;
  }
}

.address-list {
  cursor: pointer;
}

.btn.btn-primary {
  background-color: #001632;
}

.btn.btn-primary:hover {
  background-color: #52B363;
  color: #000;
}

.navbar {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  background-color: rgba(255, 255, 255, 0.9);
  box-shadow: 0 0 2px 0 rgba(#001632, .2), 0 5px 10px 0 rgba(#001632, .1);
  z-index: 1000;
}

.search-bar-container {
  transition: margin-top 0.3s ease-in-out;
}

.advanced-search-open .search-bar-container {
  margin-top: 10rem;
}

img {
  height: 35px;
}

.advanced-search-container {
  position: fixed;
  top: 380%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 1001;
  /* Adjust z-index as needed */
}

.navbar-nav {
  display: flex;
  justify-content: space-between;
}

.navbar-nav a {
  position: relative;
  padding: 15px;
  text-decoration: none;
  color: #001632;
}

.navbar-nav a::before {
  content: "";
  position: absolute;
  left: 0;
  right: 100%;
  top: 0;
  height: 3px;
  background-color: #52B363;
  transition: right 0.3s ease;
}

.navbar-nav a:hover::before {
  right: 0;
}

.navbar-nav a:hover {
  color: #001632;
}

.navbar-nav .active-link::before {
  right: 0;
}

.navbar-nav .active-link {
  color: #001632;
  font-weight: bold;
}

.font-weight-bold {
  font-weight: bold;
}

.mini-p {
  font-size: 0.8rem;
  color: #757575;
}

.button-container {
  position: absolute;
  bottom: 10px;
  right: 10px;
}

.btn-apply-filters {
  background-color: #001632;
  color: #fff;
  transition: background-color 0.3s ease;

  &:hover {
    background-color: #52B363;
  }
}

.apartment-link {
  text-decoration: none;
  color: #001632;
}

ul {
  list-style: none;
}
</style>
