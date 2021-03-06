<template>
  <div class="flex items-stretch sm:items-center justify-center min-h-screen">
    <div class="block-wrapper py-24 w-full" v-show="this.weather?.length === 0">
      <CreateCitiesList :fetchCitiesFromFile="fetchCitiesFromFile" />
    </div>
    <div
      v-show="this.error.status"
      class="bg-red-300 text-red-900 fixed top-7 left-1/2 transform -translate-x-1/2 z-20 px-3 py-1.5 rounded-lg border border-red-500"
    >
      {{ this.error.msg }}
    </div>
    <div
      class="content-wrapper z-10 bg-white relative shadow-md rounded-lg w-full mx-auto my-0 sm:my-10 sm:mx-10 px-3 sm:px-7 lg:px-4 py-10 flex flex-col lg:flex-row lg:justify-around lg:items-center max-w-7xl"
      v-show="this.weather?.length > 0"
    >
      <div class="lg:h-188">
        <UserInformation :stopAPIRefreshing="_stopAPIRefreshing" />
        <div class="mt-12">
          <div class="flex items-center justify-center lg:justify-start">
            <AddCityForm
              :submitFn="handleCityAdd"
              :singleCity="true"
              :placeholder="`Add a city to your list`"
              :cityAlreadyAdded="this.cityAlreadyAdded"
            />
          </div>
          <div class="relative sm:left-8 mt-10 flex flex-col items-center">
            <h2 class="text-sky-800 tracking-wide font-semibold text-3xl">
              Weather Forecast
            </h2>
            <div class="mt-8" v-show="this.weather">
              <h3 class="my-5 font-medium">Your cities</h3>
              <button
                @click="clearCitiesList"
                class="px-2 py-1.5 text-xs sm:text-sm sm:px-3 sm:py-1.5 bg-sky-500 rounded-md text-white block ml-auto mb-3 uppercase hover:-translate-y-1 hover:shadow-md transform transition-transform duration-300 active:transform-y-1 focus:transform-y-1 font-medium"
              >
                clear cities
              </button>
              <Table
                :data="this.weather"
                @display-hourly-data="this.showInTimeData"
              />
            </div>
          </div>
        </div>
      </div>
      <div
        class="mt-12 text-center font-medium relative lg:w-80 lg:h-167 mb-12"
      >
        <div
          v-show="this.loading"
          class="absolute left-1/2 top-1/2 transform -translate-x-1/2 -translate-y-1/2"
        >
          <Spinner />
        </div>
        <div
          class="select-none"
          v-show="!this.hourlyData?.city && !this.loading"
        >
          <h5
            class="mt-5 lg:mt-0 lg:absolute lg:top-1/2 lg:transform lg:-translate-y-1/2 font-normal text-gray-500"
          >
            Click on
            <span class="font-semibold text-sky-700 animate-pulse">More</span>
            button to see hourly weather
          </h5>
        </div>
        <div
          v-show="this.hourlyData?.city && !this.loading"
          class="mt-4 flex flex-col justify-around"
        >
          <h5 class="mt-8 mb-3 text-3xl text-cyan-600">
            {{ this.hourlyData?.city || '' }}
          </h5>
          <span class="text-lg text-blue-500 mb-5"
            >{{ this.hourlyData?.temp?.toFixed(1) }} °C</span
          >
          <div class="mt-4 flex flex-col justify-around">
            <LineChart :chartData="lineChartData" title="Temperature" />

            <BarChart :data="this.hourlyData.weather" chartTitle="Humidity" />
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import citiesJson from '@/dev-data/city.list.json';
import CreateCitiesList from '@/components/CreateCitiesList/CreateCitiesList';
import UserInformation from '@/components/UserInformation/UserInformation';
import AddCityForm from '@/components/Forms/AddCityForm';
import Table from '@/components/Table/Table';
import BarChart from '@/components/BarChart/BarChart';
import LineChart from '@/components/LineChart/LineChart';
import Spinner from '@/components/Spinner/Spinner';
import { API_REFRESH_RATE } from '@/constants';

import {
  formatHourlyData,
  storageFn,
  attachWeatherToCities,
  apiRequests,
  buildLineChartData,
} from '@/utils';

export default {
  name: 'WeatherPage',
  components: {
    CreateCitiesList,
    UserInformation,
    AddCityForm,
    Table,
    BarChart,
    LineChart,
    Spinner,
  },
  data() {
    return {
      loading: false,
      observedCities: [],
      weather: [],
      lineChartData: {},
      hourlyData: [],
      citiesFounded: false,
      cityAlreadyAdded: false,
      error: {
        status: false,
        msg: '',
      },
      intervalId: null,
    };
  },
  mounted() {
    const storageData = storageFn.getStorageData('weatherData');
    if (storageData?.length > 0) {
      this.observedCities = storageData;
      this.getAPIData();
      this.intervalId = setInterval(
        () => this.getAPIData(),
        API_REFRESH_RATE * 1000
      );
    }
  },
  methods: {
    _stopAPIRefreshing() {
      clearInterval(this.intervalId);
    },

    _showHideError(msg) {
      this.error = { status: true, msg };
      setTimeout(() => {
        this.error = { status: false, msg: '' };
      }, 2000);
    },

    _scrollTableToView(element) {
      setTimeout(() => (element.scrollTop = element.scrollHeight + 57), 500);
    },

    clearCitiesList() {
      this.weather = [];
      this.observedCities = [];
      this.hourlyData = {};
      clearInterval(this.intervalId);
      storageFn.removeStorageItem('weatherData');
    },

    async handleCityAdd(cityName, isCityInvalid) {
      try {
        if (isCityInvalid) {
          this._showHideError('Please provide a valid city name!');
          return;
        }
        if (this.weather.some(cityData => cityData.name === cityName.trim())) {
          this._showHideError('This city is already on your list!');
          return;
        }

        const loadedCity = await this.fetchCitiesFromFile(
          [cityName.trim()],
          false
        );

        if (!loadedCity) {
          this._showHideError(
            "Sorry, but this country doesn't exist in our database!"
          );
          return;
        }

        this.observedCities = [...this.observedCities, loadedCity];

        await this.getAPIData();

        this._stopAPIRefreshing();

        this.intervalId = setInterval(
          () => this.getAPIData(),
          API_REFRESH_RATE * 1000
        );

        setTimeout(() => {
          this.$el.querySelector('ul').scrollTop =
            this.$el.querySelector('ul').scrollHeight;
        }, 500);
      } catch (err) {
        this._showHideError('Sorry but we cannot load the city');
      }
    },

    async showInTimeData(lat, lon, city) {
      try {
        this.loading = true;
        const dataOverTime = await apiRequests.renderRequestForInTimeData(
          lat,
          lon
        );

        const { temp } = this.weather.find(data => data.name === city);

        this.hourlyData = {
          city,
          temp,
          weather: formatHourlyData(dataOverTime.data.hourly.slice(0, 12)),
        };

        this.lineChartData = buildLineChartData(this.hourlyData.weather);
      } catch (err) {
        this._showHideError('Sorry but we could not load the data!');
      } finally {
        this.loading = false;
      }
    },

    async getAPIData() {
      try {
  
        await Promise.all(
          apiRequests.renderRequestsForCities(this.observedCities)
        ).then(citiesWeather => {
          this.weather = attachWeatherToCities(
            citiesWeather,
            this.observedCities
          );
        });
      } catch (err) {
        if (err.message.includes('undefined')) return;
        this._showHideError('We could not load the resources');
      }
    },

    async fetchCitiesFromFile(cities, manyCities = true) {
      let returnedCity = {};
      cities.forEach(cityName => {
        for (let i = 0, arrLen = citiesJson.length; i < arrLen; i++) {
          if (citiesJson[i].name.toLowerCase() === cityName.toLowerCase()) {
            if (manyCities) {
              this.observedCities = [...this.observedCities, citiesJson[i]];
              this.citiesFounded = true;
            }

            if (!manyCities) {
              this.citiesFounded = true;
              returnedCity = citiesJson[i];
              storageFn.setStorageData('weatherData', [
                ...this.observedCities,
                citiesJson[i],
              ]);
            }
            break;
          }
        }
      });

      if (!this.citiesFounded)
        this._showHideError(
          `Sorry but our database does not include selected ${
            cities.length > 1 ? 'cities' : 'city'
          }`
        );

      try {
        if (manyCities) {
          await this.getAPIData().then(() => {
            if (this.observedCities.length !== cities.length) {
              this._showHideError(
                'Sorry, but we could not load all the cities.'
              );
            }

            storageFn.setStorageData('weatherData', this.observedCities);
            this.intervalId = setInterval(
              () => this.getAPIData(),
              API_REFRESH_RATE * 1000
            );
          });
        }
      } catch (err) {
        this._showHideError('Some unexpected error occurred!');
      }

      if (Object.keys(returnedCity).length > 0) return returnedCity;
    },
  },
};
</script>

<style scoped>
.block-wrapper {
  min-height: 100vh;
}

.content-wrapper {
  min-height: 750px;
}
</style>
