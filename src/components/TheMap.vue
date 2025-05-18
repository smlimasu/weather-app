<script setup lang="ts">
import { ref, watch } from 'vue'
import TheWeather from './TheWeather.vue'
import { ArcgisMap } from '@arcgis/map-components/components/arcgis-map'
import { ArcgisZoom } from '@arcgis/map-components/components/arcgis-zoom'
import { ArcgisPlacement } from '@arcgis/map-components/components/arcgis-placement'
import { ArcgisLocate } from '@arcgis/map-components/components/arcgis-locate'
import type { ArcgisMapCustomEvent, TargetedEvent } from '@arcgis/map-components'
import FeatureLayer from '@arcgis/core/layers/FeatureLayer'

const OPENCAGE_KEY = import.meta.env.VITE_OPENCAGE_API_KEY

// --- Reactive location data ---
const locationLatitude = ref<string | null>(null)
const locationLongitude = ref<string | null>(null)
const cityName = ref<string | null>(null)

watch([locationLatitude, locationLongitude], async ([lat, lon]) => {
  if (lat && lon) {
    cityName.value = await getCityName(lat, lon)
  } else {
    cityName.value = null
  }
})

// --- Renderer setup ---

/**
 * Renderer configuration for displaying city features on the map.
 * Autocasts to a `SimpleRenderer` object.
 *
 * Properties:
 * - type: Specifies the renderer type - ('simple').
 * - symbol: Defines the marker's appearance:
 *   - type: The symbol type ('simple-marker').
 *   - size: The diameter of the marker in points.
 *   - color: The fill color of the marker.
 *   - outline: The outline of the marker, including width and color.
 */
const worldCitiesRenderer = {
  type: 'simple' as const,
  symbol: {
    type: 'simple-marker' as const,
    size: 6,
    color: 'black',
    outline: {
      width: 0.5,
      color: 'white',
    },
  },
}

/**
 * Size visual variable configuration for the renderers map symbols.
 * Autocasts to a `SizeVariable` object.
 *
 * This object defines how the size of map symbols should vary based on the 'POP' (population) field.
 * - `type`: Specifies the visual variable type, set to 'size'.
 * - `field`: The data field used to determine symbol size ('POP').
 * - `minDataValue`: The minimum data value corresponding to the smallest symbol size.
 * - `maxDataValue`: The maximum data value corresponding to the largest symbol size.
 * - `minSize`: The minimum symbol size.
 * - `maxSize`: The maximum symbol size.
 */
const worldCitiesSizeVisualVariable = {
  type: 'size',
  field: 'POP',
  minDataValue: 1000000,
  maxDataValue: 14000000,
  minSize: 4,
  maxSize: 40,
}

/**
 * Defines a new FeatureLayer for displaying world cities.
 *
 * - `url`: Points to the ArcGIS FeatureServer containing world cities data.
 * - `renderer`: Uses the `worldCitiesRenderer` object to define how features are drawn.
 * - `visualVariables`: Applies the `worldCitiesSizeVisualVariable` to dynamically set symbol sizes based on attribute values.
 *
 * This layer is used by the MapView to visualize global city locations with customized rendering.
 */
const worldCitiesLayer = new FeatureLayer({
  // optimization notice: the various layers, renderers etc. could be moved to a separate module/base config file/globals to enhance reusability and maintainability
  url: 'https://services.arcgis.com/P3ePLMYs2RVChkJx/ArcGIS/rest/services/World_Cities/FeatureServer/0',
  renderer: {
    ...worldCitiesRenderer,
    visualVariables: [worldCitiesSizeVisualVariable],
  },
})

/**
 * Listens to the the map ready event and is then used to add the `worldCitiesLayer` to the ArcGIS map elements instance.
 *
 * @param {ArcgisMapCustomEvent<HTMLArcgisMapElement>} event - The custom event triggered when the map is ready.
 */
const onMapReady = async (event: ArcgisMapCustomEvent<HTMLArcgisMapElement>) => {
  const arcgismap = event.target
  arcgismap.addLayer(worldCitiesLayer)
}

/**
 * Retrieves the city name in german lang for the given latitude and longitude using the OpenCage Geocoding API (reverse geocode).
 *
 * @param {string} lat - The latitude coordinate as a string.
 * @param {string} lon - The longitude coordinate as a string.
 * @returns {Promise<string>} The resolved city name, or 'n.V.' if not available, or an error message if the request fails.
 *
 */
const getCityName = async (lat: string, lon: string): Promise<string> => {
  // optimization notice: should be refactored to properly typed method in own service/module
  const url = new URL('https://api.opencagedata.com/geocode/v1/json')
  url.searchParams.set('key', OPENCAGE_KEY)
  url.searchParams.set('q', `${lat},${lon}`)
  url.searchParams.set('language', 'de')
  url.searchParams.set('no_annotations', '1')
  url.searchParams.set('limit', '1')

  try {
    //an alternative would be to use vue tools like useFetch, but this is a simple fetch request (avoid unnecessary dependencies)
    const response = await fetch(url.toString())
    const data = await response.json()
    const components = data.results?.[0]?.components || {}
    //using normalized_city prop to account for different concepts of human settlement like town, city, village, etc. - edge cases like ocean locations return 'n.V.'
    return (components._normalized_city || components.city) ?? 'n.V.'
  } catch (error) {
    console.error('Error fetching city name', error)
    return 'error fetching city name'
  }
}

/**
 * Handles the users map view click event.
 * Extracts the latitude and longitude from the clicked map point and updates the corresponding reactive values.

 * If the coordinates are not available, sets them to 'n.V.' (not available).
 *
 * @param {TargetedEvent<ArcgisMap, __esri.ViewClickEvent>} event - The click event containing map point details.
 */
const onViewClick = (event: TargetedEvent<ArcgisMap, __esri.ViewClickEvent>) => {
  const mapPoint = event.detail?.mapPoint
  const { latitude, longitude, spatialReference } = mapPoint

  // Check if the spatial reference is Web Mercator (wkid: 102100)
  // Since the spatial reference of the basemap is Web Mercator, the latitude property will be given in WGS84, otherwise it would be y/x. The implementation of a transformation function is theoretically missing, but not necessary in this case.
  if (!spatialReference.isWebMercator) {
    console.warn('coordinates spatial reference not WGS84:', spatialReference)
    return
  }

  if (!latitude || !longitude) {
    locationLatitude.value = 'n.V.'
    locationLongitude.value = 'n.V.'
  } else {
    locationLatitude.value = latitude.toFixed(3)
    locationLongitude.value = longitude.toFixed(3)
  }
}
</script>

<template>
  <arcgis-map
    basemap="gray-vector"
    center="24,57"
    zoom="3"
    @arcgisViewReadyChange="onMapReady"
    @arcgisViewClick="onViewClick"
  >
    <arcgis-zoom position="top-left" />
    <arcgis-locate position="bottom-left" scale="5000" />
    <arcgis-placement position="top-right">
      <aside class="widget-container">
        <section class="coordinates-info-container">
          <h3>Getroffene Koordinaten:</h3>
          <template v-if="locationLatitude && locationLongitude">
            <p>Longitude: {{ locationLongitude }}</p>
            <p>Latitude: {{ locationLatitude }}</p>
          </template>
          <template v-else>
            <i>Klicke auf einen Ort in der Karte, um die Koordinaten anzuzeigen. </i>
          </template>
        </section>
        <TheWeather :cityName="cityName" />
      </aside>
    </arcgis-placement>
  </arcgis-map>
</template>

<style scoped>
.widget-container {
  display: flex;
  flex-direction: column;
  gap: var(--calcite-spacing-md);
  height: 95vh;
  align-items: flex-end;
}
.coordinates-info-container {
  font-size: var(--calcite-font-size);
  border-radius: var(--calcite-corner-radius-round);
  background-color: var(--calcite-color-foreground-1);
  padding: var(--calcite-spacing-md);
  font-family: var(--calcite-font-family);
  border-color: var(--calcite-color-border-1);
  display: inline-flex;
  flex-direction: column;
  justify-content: space-between;
  gap: 5px;
  inline-size: clamp(200px, 50%, 400px);
  box-shadow: var(--calcite-shadow-sm);

  h3,
  p,
  i {
    margin: 0;
    font-size: inherit;
    color: var(--calcite-color-text-1);
  }
}
</style>
