<!-- eslint-disable @typescript-eslint/no-explicit-any -->

<script setup lang="ts">
import { ref, computed, watchEffect, onMounted } from 'vue'

const OPENWEATHER_KEY = import.meta.env.VITE_OPENWEATHER_API_KEY

// --- Types ---
interface WeatherEntry {
  dt: number
  dt_txt: string
  main: {
    temp: number
    humidity: number
  }
  weather: Array<{
    main: string
    description: string
    icon: string
  }>
  wind?: {
    speed: number
  }
  pop?: number
}

interface DailyWeather {
  date: string
  weatherEntries: WeatherEntry[]
}

interface LocationInfo {
  name: string
  country: string
}

// --- component inputs ---
const props = defineProps<{
  cityName: string | null
}>()

// --- Reactive data ---
const locationInfoData = ref<LocationInfo | null>(null)
const completeWeatherData = ref<WeatherEntry[]>([])
const dailyWeatherData = ref<DailyWeather[]>([])

const isLoading = ref(false)
const error = ref<string | null>(null)
const lastFetchedCity = ref<string | null>(null)

/**
 * Set of selected timestamps to display the weather for (e.g. 18:00)
 */
const targetHours = [6, 12, 18, 21]

// Using a small cache to prevent repeated fetches for the same city
const weatherCache = ref<Map<string, { timestamp: number; data: any }>>(new Map())
const CACHE_DURATION = 10 * 60 * 1000 // 10 minutes

const resetData = () => {
  locationInfoData.value = null
  dailyWeatherData.value = []
  completeWeatherData.value = []
}

/**
 * Fetches the weather forecast for the next 5 days from the openweathermap api.
 *
 * @param cityName - The city to fetch the weather for.
 */
const fetchWeatherForecast = async (cityName: string) => {
  //Optimization notice: results would be more reliable if the lat, long coordinates could (also) be used to query the e.g. nearest city or to distinguish e.g. locations in the middle of the ocean.
  if (!cityName) return

  // Prevent duplicate fetches
  if (cityName === lastFetchedCity.value) return

  // Check cache first
  const cachedData = weatherCache.value.get(cityName)
  if (cachedData && Date.now() - cachedData.timestamp < CACHE_DURATION) {
    processWeatherData(cachedData.data)
    return
  }

  error.value = null
  isLoading.value = true
  lastFetchedCity.value = cityName

  try {
    const url = new URL('https://api.openweathermap.org/data/2.5/forecast')
    url.searchParams.set('q', cityName)
    url.searchParams.set('appid', OPENWEATHER_KEY)
    url.searchParams.set('units', 'metric')
    url.searchParams.set('lang', 'de')

    const res = await fetch(url.toString())
    const data = await res.json()

    if (!res.ok) {
      throw new Error(data.message || 'Fehler beim Abrufen der Wetterdaten')
    }

    // Cache the result
    weatherCache.value.set(cityName, {
      timestamp: Date.now(),
      data,
    })

    processWeatherData(data)
  } catch (err) {
    console.error('Forecast error:', err)
    error.value = err instanceof Error ? err.message : 'Unbekannter Fehler beim Laden der Daten'
    resetData()
  } finally {
    isLoading.value = false
  }
}

/**
 * Processes the raw weather data and transforms it into a format suitable for the application.
 * @param {any} data - The raw weather data to be processed.
 * @returns {any} The processed weather data.
 */
const processWeatherData = (data: any) => {
  if (!data || !data.list || !data.city) {
    error.value = 'Ungültige Wetterdaten erhalten'
    resetData()
    return
  }

  completeWeatherData.value = data.list

  //Convert unix timestamps to Dates, group data in a group for every day and filter out unwanted timeslots for easier iteration
  const preparedData: Record<string, WeatherEntry[]> = data.list.reduce(
    (groups: any, entry: WeatherEntry) => {
      const date = new Date(entry.dt * 1000)
      const dateString = date.toDateString()

      if (!groups[dateString]) groups[dateString] = []
      if (targetHours.includes(date.getUTCHours())) groups[dateString].push(entry)
      return groups
    },
    {},
  )

  dailyWeatherData.value = Object.entries(preparedData).map(([date, weatherEntries]) => ({
    date,
    weatherEntries,
  }))

  locationInfoData.value = {
    name: data.city.name,
    country:
      new Intl.DisplayNames(['de'], { type: 'region' }).of(data.city.country) || data.city.country,
  }
}

// Computed properties
// Get first item as current weather (e.g. when current time is 14:30, the first item is 15:00)
const currentWeather = computed(() => completeWeatherData.value[0] || null)
const todayBlocks = computed(() => dailyWeatherData.value[0]?.weatherEntries ?? [])
const nextDays = computed(() => dailyWeatherData.value.slice(1))

// Format a date based on the provided options
const formatDate = (dateStr: string, options: Intl.DateTimeFormatOptions) => {
  return new Date(dateStr).toLocaleDateString('de-DE', options)
}

// Format time in locale format
const formatTime = (dateStr: string) => {
  return new Date(dateStr).toLocaleTimeString('de-DE', { timeStyle: 'short' })
}

// Watch for city name changes
watchEffect(() => {
  if (props.cityName) {
    fetchWeatherForecast(props.cityName)
  }
})

// Clean up excessive cache entries
const cleanupCache = () => {
  if (weatherCache.value.size > 10) {
    // Keep only the 5 most recent entries
    const entries = Array.from(weatherCache.value.entries())
      .sort((a, b) => b[1].timestamp - a[1].timestamp)
      .slice(0, 5)

    weatherCache.value = new Map(entries)
  }
}

onMounted(() => {
  // Set up periodic cache cleanup
  const interval = setInterval(cleanupCache, 30 * 60 * 1000) // Every 30 minutes

  return () => clearInterval(interval)
})
</script>

<!-- Optimization notice: The following elements should be refactored into smaller, separate components incl. their scss for better readability and maintainability, e.g.:
  - DayWeatherCard
  - WeatherTimeBlock
  - Loading state
  - Error, Warning state components
-->
<template>
  <section class="weather-container" aria-label="Wettervorhersage">
    <!-- Loading state -->
    <div v-if="isLoading" class="loading-state" role="status" aria-live="polite">
      <calcite-loader scale="m" text="Lade Wetterdaten..."></calcite-loader>
      <span>Lade Wetterdaten...</span>
    </div>

    <!-- Error state -->
    <div v-else-if="error" class="error-state" role="alert">
      <calcite-icon icon="exclamation-mark-triangle-f" scale="m" />
      <p>{{ error }}</p>
    </div>

    <!-- Weather data -->
    <template v-else-if="completeWeatherData.length && locationInfoData">
      <div class="location">
        <calcite-icon icon="map-pin" />
        <h2>{{ locationInfoData.name }}, {{ locationInfoData.country }}</h2>
      </div>
      <!-- Today's weather data -->
      <div class="current-card">
        <div v-if="currentWeather" class="today-info">
          <p class="day-label">
            Heute ({{ formatDate(currentWeather.dt_txt, { weekday: 'long' }) }})
          </p>
          <p class="now-label">Aktuelles Wetter</p>

          <div class="current-weather">
            <span class="main-temp" aria-label="Temperatur"
              >{{ Math.round(currentWeather.main.temp) }}°</span
            >
            <img
              :src="`https://openweathermap.org/img/wn/${currentWeather.weather[0].icon}@2x.png`"
              :alt="currentWeather.weather[0].description"
              class="weather-icon"
            />
            <div class="weather-details">
              <p>{{ currentWeather.weather[0].description }}</p>
              <p class="extra-info">
                Luftfeuchtigkeit:
                {{ currentWeather.main.humidity }}%
              </p>
              <p v-if="currentWeather.wind" class="extra-info">
                Wind: {{ Math.round(currentWeather.wind.speed * 3.6) }} km/h
              </p>
            </div>
          </div>

          <div class="blocks-grid">
            <div
              v-for="entry in todayBlocks"
              :key="entry.dt"
              class="time-block"
              :aria-label="`${formatTime(entry.dt_txt)} - ${Math.round(entry.main.temp)}°C, ${entry.weather[0].description}`"
            >
              <span class="temp-bold">{{ Math.round(entry.main.temp) }}°C</span>
              <img
                :src="`https://openweathermap.org/img/wn/${entry.weather[0].icon}@2x.png`"
                :alt="entry.weather[0].description"
                class="weather-icon-small"
              />
              <span class="time-label">
                {{ formatTime(entry.dt_txt) }}
              </span>
              <span v-if="entry.pop" class="rain-info">
                <calcite-icon icon="water-drop" scale="s" />{{ entry.pop * 100 }}%
              </span>
            </div>
          </div>
        </div>
      </div>
      <!-- 5-Day-Forecast weather data -->
      <span class="forecast-title-container">
        <calcite-icon icon="calendar" scale="m" />
        <h3 class="forecast-title">5-Tage-Wettervorhersage</h3>
      </span>

      <section class="forecast-section">
        <div v-for="day in nextDays" :key="day.date" class="day-block">
          <h4 class="day-heading">
            {{ formatDate(day.date, { dateStyle: 'full' }) }}
          </h4>
          <div class="blocks-grid">
            <div
              class="time-block"
              v-for="entry in day.weatherEntries"
              :key="entry.dt"
              :aria-label="`${formatTime(entry.dt_txt)} - ${Math.round(entry.main.temp)}°C, ${entry.weather[0].description}`"
            >
              <span class="temp-bold">{{ Math.round(entry.main.temp) }}°C</span>
              <img
                :src="`https://openweathermap.org/img/wn/${entry.weather[0].icon}@2x.png`"
                :alt="entry.weather[0].description"
                class="weather-icon-small"
              />
              <span class="time-label">
                {{ formatTime(entry.dt_txt) }}
              </span>
              <span v-if="entry.pop" class="rain-info">
                <calcite-icon icon="water-drop" scale="s" />{{ entry.pop * 100 }}%
              </span>
            </div>
          </div>
        </div>
      </section>
    </template>

    <!-- No data state -->
    <div v-else class="empty-state">
      <calcite-icon icon="exclamation-mark-triangle-f" />
      <p>Keine Wetterdaten verfügbar</p>
    </div>
  </section>
</template>

<style scoped>
.weather-container {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  padding: var(--calcite-spacing-md);
  background: var(--calcite-color-background);
  border-radius: var(--calcite-corner-radius-round);
  box-shadow: var(--calcite-shadow-sm);
  width: 400px;
  font-family: var(--calcite-font-family);
  overflow: hidden;
}

p,
h1,
h2,
h3,
h4 {
  font-size: var(--calcite-font-size);
  margin: 0;
}

.location,
.forecast-title-container {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  --calcite-font-size: var(--calcite-font-size-lg);
}

.today-info {
  background: white;
  padding: 1rem;
  border-radius: var(--calcite-corner-radius-round);
  box-shadow: var(--calcite-shadow-sm);
  display: flex;
  flex-direction: column;
}

.day-label {
  font-size: var(--calcite-font-size-md);
  font-weight: bold;
  margin: 0;
}

.current-weather {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  margin: 0;
}

.main-temp {
  font-size: 2.5rem;
  font-weight: bold;
}

.weather-icon {
  filter: drop-shadow(0 2px 4px rgba(0, 0, 0, 0.25));
  margin-bottom: -30px;
  margin-top: -30px;
}

.weather-icon-small {
  width: 2rem;
  height: 2rem;
}

.weather-details {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.extra-info {
  font-size: var(--calcite-font-size-sm);
  color: var(--calcite-color-text-3);
}

.time-block {
  display: flex;
  flex-direction: column;
  align-items: center;
  box-shadow: var(--calcite-shadow-sm);
  padding: 0.5rem;
  border-radius: var(--calcite-corner-radius-round);
  transition: transform 0.3s ease;
}

.time-block:hover {
  transform: translateY(-2px);
  box-shadow: var(--calcite-shadow-sm);
}

.temp-bold {
  font-weight: bold;
}

.time-label {
  font-size: 0.75rem;
  color: rgb(141, 141, 141);
  font-weight: bold;
}

.blocks-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(80px, 1fr));
  gap: 0.5rem;
  margin-top: 0.5rem;
}

.day-block {
  background: white;
  padding: 1rem;
  border-radius: var(--calcite-corner-radius-round);
  box-shadow: var(--calcite-shadow-sm);
  margin-right: 0.5rem;
}

.day-heading {
  margin-bottom: 0.5rem;
  font-size: var(--calcite-font-size-md);
}

.forecast-section {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  overflow: auto;
}

.forecast-title {
  margin: 0;
  font-size: var(--calcite-font-size-md);
}

.rain-info {
  font-size: 0.7rem;
  color: #86aadd;
}

.loading-state,
.error-state,
.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  padding: var(--calcite-spacing-md);
  box-shadow: var(--calcite-shadow-sm);
  border-radius: var(--calcite-corner-radius-round);
  font-style: italic;
  --calcite-icon-color: var(--calcite-color-status-warning);
}

.error-state {
  color: var(--calcite-color-status-danger);
  --calcite-icon-color: var(--calcite-color-status-danger);
}
</style>
