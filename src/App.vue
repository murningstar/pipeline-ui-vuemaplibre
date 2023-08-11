<script setup lang="ts">
import { CFormCheck } from '@coreui/vue'

import maplibregl, { Map as MapLibreMap, Marker, Popup, type StyleSpecification } from 'maplibre-gl'; // or "const maplibregl = require('maplibre-gl');"
import { onMounted, onBeforeUnmount, ref, watchEffect, watch, reactive } from 'vue';
import { io } from "socket.io-client";
import axios from "axios"

type StoreData = {
  storeId: string,
  name?: string,
  featureObj?: Object,
  range?: Object,
  total?: string,
  max?: string,
}
type UpdateDTO = {
  storeId: string,
  value: string
}
type CurrentMode = "total" | "max"


const socket = io("http://localhost:7337");


const retailStoresData = ref<Map<string, StoreData>>(new Map())


let map: MapLibreMap | null = null
const markersAndPopups = ref<Map<string, { markerColor: string, markerObj: Marker, popupObject?: Popup }>>(new Map())
const defaultMarkerColor = '#3fb1ce'
const highestMarkerColor = '#d95635'


/* map & storesInfo from 'Generator' service load state */
const loadState = reactive({
  storesInfoLoaded: false,
  mapLoaded: false,
  cachedTotalInitialized: false,
  cachedMaxInitialized: false,
  isFirstLoad: true,
})

/* `loadState` watch effect creating markers */
watch(loadState, (newValue) => {
  if (newValue.storesInfoLoaded && newValue.mapLoaded && newValue.isFirstLoad) {
    loadState.isFirstLoad = false
    createMarkers()
    if (newValue.cachedTotalInitialized && newValue.cachedMaxInitialized) {
      updateMarkerByStoreId(highestAccountingObj.totalId!, highestMarkerColor) // вызов после загрузки необходим, т.к. fetch происходит в setup(), т.е. обычно еще до создания маркеров
      createPopups()
    }
  }
})


/* Current highlighed marker (by max sold receipt / by total sales) */
const currentMode = ref<CurrentMode>("total")

/* `currentMode`'s watch effect for updating markers' */
watch(() => currentMode.value, (newValue) => {
  if (newValue == "max") {
    updateMarkerByStoreId(highestAccountingObj.totalId!, defaultMarkerColor)
    updateMarkerByStoreId(highestAccountingObj.maxId!, highestMarkerColor)
  }
  if (newValue == "total") {
    updateMarkerByStoreId(highestAccountingObj.maxId!, defaultMarkerColor)
    updateMarkerByStoreId(highestAccountingObj.totalId!, highestMarkerColor)
  }
})


const highestAccountingObj = reactive<{
  maxId?: string,
  maxValue?: number,
  totalId?: string,
  totalValue?: number
}>({})

/* `highestAccountingObj`'s watch effect for updating markers */
watch([() => highestAccountingObj.maxId, () => highestAccountingObj.totalId], ([newMaxId, newTotalId], [oldMaxId, oldTotalId]) => {
  /* проверка, инициализированы ли вообще значения до этого, т.к. они будут использоваться */
  if (oldMaxId && oldTotalId) {

    /* Если обновился highestMax */
    if (newMaxId != oldMaxId && currentMode.value == 'max') {
      console.log('highestMax');
      updateMarkerByStoreId(newMaxId!, highestMarkerColor)
      updateMarkerByStoreId(oldMaxId, defaultMarkerColor)

    }
    /* Если обновился highestTotal */
    if (newTotalId != oldTotalId && currentMode.value == "total") {
      console.log('highestTotal');
      updateMarkerByStoreId(newTotalId!, highestMarkerColor)
      updateMarkerByStoreId(oldTotalId, defaultMarkerColor)

    }

  }
})


/* call only when map is loaded and data fetched */
function createMarkers() {
  retailStoresData.value.forEach((store, storeId) => {
    const marker = new maplibregl.Marker({ color: defaultMarkerColor }).setLngLat(store.featureObj!.geometry.coordinates).addTo(map!)
    markersAndPopups.value.set(storeId, { markerColor: defaultMarkerColor, markerObj: marker })
  })
}
function createPopups() {
  markersAndPopups.value.forEach((value, id) => {
    const popup = new maplibregl.Popup({ closeOnClick: false })
      .setLngLat(value.markerObj.getLngLat())
    popup.setHTML(
      `<p> Store ID: ${id} </p>
          <p> Max receipt: ${retailStoresData.value.get(id)?.max} </p>
          <p> Total sales: ${retailStoresData.value.get(id)?.total} </p>`
    )
    popup.addTo(map!)
    const popupContentElement = (popup.getElement().querySelectorAll(`[class="maplibregl-popup-content mapboxgl-popup-content"]`)[0] as HTMLElement)
    const popupTipElement = (popup.getElement().querySelectorAll(`[class="maplibregl-popup-tip mapboxgl-popup-tip"]`)[0] as HTMLElement)
    popupContentElement.classList.add('my-popup-content') // main.css
    popupTipElement.classList.add('my-popup-tip') // main.css

    value.markerObj.setPopup(popup)
  })
}

/* color in "#ffffff" format */
function updateMarkerByStoreId(storeId: string, color: string) {
  const marker = markersAndPopups.value.get(storeId)!
  const htmlElement = (marker.markerObj._element.querySelectorAll(`[fill="${marker.markerColor}"]`)[0] as SVGGElement)
  marker.markerColor = color
  htmlElement.setAttribute('fill', color)
  htmlElement.style.fill = color
  markersAndPopups.value.set(storeId, marker)
}
function updatePopup(updateData: UpdateDTO) {
  const popup = markersAndPopups.value.get(updateData.storeId)?.markerObj.getPopup()!
  popup.setHTML(
    `<p> Store ID: ${updateData.storeId} </p>
            <p> Max receipt: ${retailStoresData.value.get(updateData.storeId)?.max} </p>
            <p> Total sales: ${retailStoresData.value.get(updateData.storeId)?.total} </p>`
  )
}

// function removeMarkers() {
//   markersAndPopups.value.forEach((marker) => {
//     marker.remove()
//   })
//   markersAndPopups.value.splice(0, markersAndPopups.value.length)
// }

/* fetch stores info from 'Generator' service and put it into retailStoresData map*/
async function initRetailStoresData() {
  const storesData: StoreData[] = await (await axios.get('http://localhost:7331/stores')).data
  console.log("storesData(axios): ", storesData);
  storesData.forEach(storeData => {
    let store: StoreData | null = null
    if (retailStoresData.value.has(storeData.storeId)) {
      store = retailStoresData.value.get(storeData.storeId)!
    }
    if (!retailStoresData.value.has(storeData.storeId)) {
      store = { storeId: storeData.storeId }
    }
    Object.keys(storeData).forEach(key => {
      store[key] = storeData[key]
    })
    retailStoresData.value.set(storeData.storeId, store!)
  })
  loadState.storesInfoLoaded = true
}
initRetailStoresData()

/* called in socket.io 'totalCached' callback */
function initCachedTotal(redisCache: {}) {
  Object.entries<string>(redisCache).forEach((pairOfIdAndResult) => {
    let store: StoreData | null = null
    const storeInitialized = retailStoresData.value.has(pairOfIdAndResult[0])
    if (storeInitialized) {
      store = retailStoresData.value.get(pairOfIdAndResult[0])!
    } else {
      store = { storeId: pairOfIdAndResult[0] }
    }
    store!.total = pairOfIdAndResult[1]
    retailStoresData.value.set(pairOfIdAndResult[0], store!)
  })
  setInitialHighestTotal()
  loadState.cachedTotalInitialized = true
}
/* called in socket.io 'maxCached' callback */
function initCachedMax(redisCache: {}) {
  Object.entries<string>(redisCache).forEach((pairOfIdAndResult) => {
    let store: StoreData | null = null
    const storeInitialized = retailStoresData.value.has(pairOfIdAndResult[0])
    if (storeInitialized) {
      store = retailStoresData.value.get(pairOfIdAndResult[0])!
    } else {
      store = { storeId: pairOfIdAndResult[0] }
    }
    store!.max = pairOfIdAndResult[1]
    retailStoresData.value.set(pairOfIdAndResult[0], store!)
  })
  setInitialHighestMax()
  loadState.cachedMaxInitialized = true
}
/* initialize `highestTotalAccounter.highestTotal` (init by initCachedTotal)*/
function setInitialHighestTotal() {
  retailStoresData.value.forEach((store) => {
    const currentTotal = highestAccountingObj.totalValue
    /* Если highest total уже инициализирован (если не первая итерация) */
    if (currentTotal) {
      /* Если total у store с текущей итерации больше, то обновляем */
      if (parseInt(store.total!) > currentTotal) {
        highestAccountingObj.totalId = store.storeId
        highestAccountingObj.totalValue = parseInt(store.total!)
      }
    }
    /* Если не инициализирован */
    else {
      highestAccountingObj.totalId = store.storeId
      highestAccountingObj.totalValue = parseInt(store.total!)
    }
  })
}
/* initialize `highestMaxAccounter.highestMax` (init by initCachedMax))*/
function setInitialHighestMax() {
  retailStoresData.value.forEach((store) => {
    /* Если highest max уже инициализирован (если не первая итерация) */
    if (highestAccountingObj.maxValue) {
      /* Если max у store с текущей итерации больше, то обновляем */
      if (parseInt(store.max!) > highestAccountingObj.maxValue) {
        highestAccountingObj.maxId = store.storeId
        highestAccountingObj.maxValue = parseInt(store.max!)
        console.log("store max: ", store.max);
      }
    }
    /* Если не инициализирован */
    else {
      highestAccountingObj.maxId = store.storeId
      highestAccountingObj.maxValue = parseInt(store.max!)
    }
  })
}
function onUpdateSetHighestMaxIfSo(dataObj: UpdateDTO) {
  /* If highestMax is already initiated */
  if (highestAccountingObj.maxId) {
    /* If new value > highest max */
    if (parseInt(dataObj.value) > highestAccountingObj.maxValue!) {
      highestAccountingObj.maxId = dataObj.storeId
      highestAccountingObj.maxValue = parseInt(dataObj.value)
    }
  }
  /* If highestMax isn't initiated */
  else {
    highestAccountingObj.maxId = dataObj.storeId
    highestAccountingObj.maxValue = parseInt(dataObj.value)
  }
}
function onUpdateSetHighestTotalIfSo(dataObj: UpdateDTO) {
  /* If highestTotal is already initiated */
  if (highestAccountingObj.totalId) {
    /* If new value > highest Total */
    if (parseInt(dataObj.value) > highestAccountingObj.totalValue!) {
      highestAccountingObj.totalId = dataObj.storeId
      highestAccountingObj.totalValue = parseInt(dataObj.value)
    }
  }
  /* If highestTotal isn't initiated */
  else {
    highestAccountingObj.totalId = dataObj.storeId
    highestAccountingObj.totalValue = parseInt(dataObj.value)
  }
}

/* init cached total-per-store */
socket.on('totalCached', (redisCache) => {
  initCachedTotal(redisCache)
  console.log("redisCache: ", redisCache);
})
/* init cached max-per-store */
socket.on('maxCached', (redisCache) => {
  initCachedMax(redisCache)
})
/* update specific store's total */
socket.on('totalUpdate', (data: UpdateDTO) => {
  const store = retailStoresData.value.get(data.storeId)!
  store.total = data.value
  retailStoresData.value.set(data.storeId, store)
  onUpdateSetHighestTotalIfSo(data)
  updatePopup(data)
})
/* update specific store's max */
socket.on('maxUpdate', (data: UpdateDTO) => {
  const store = retailStoresData.value.get(data.storeId)!
  store.max = data.value
  retailStoresData.value.set(data.storeId, store)
  onUpdateSetHighestMaxIfSo(data)
  updatePopup(data)
})


onMounted(async () => {
  map = new maplibregl.Map({
    container: 'map', // container id
    style: 'https://demotiles.maplibre.org/style.json', // style URL
    center: [43.41994960721814, 57.57629432236976], // starting position [lng, lat]
    zoom: 4, // starting zoom
  });
  map.on('load', () => {
    loadState.mapLoaded = true
    /* map.addSource('osm-roads', {
      'type': 'vector',
      'url': 'http://localhost:3000/planet_osm_polygon'
    }); */
    /* map.addSource('osm-raster', {
      'type': 'raster',
      'url': 'https://tile.openstreetmap.org/${z}/${x}/${y}.png'
    }); */
    // Add a layer using the source
    /* map.addLayer({
      'id': 'osm-roads-layer',
      'type': 'line',
      'source': 'osm-roads',
      'source-layer': 'planet_osm_polygon',
      'layout': {},
      'paint': {
        'line-color': '#ff0000',
        'line-width': 2
      }
    }); */
  })
  map.on('click', (e) => {
    const res = map!.queryRenderedFeatures(e.point)
    console.log('longlat: ', e.lngLat);
    console.log(res);
    // updateMarkerByStoreId("2", highestMarkerColor)
  })

})
onBeforeUnmount(() => {
  socket.disconnect()
})
</script>

<template>
  <div id="map">

  </div>

  <div class="panel radio">
    <div
      v-if="loadState.storesInfoLoaded && loadState.cachedTotalInitialized && loadState.cachedMaxInitialized">
      <p>Flag store with highest: </p>
      <CFormCheck type="radio" name="radioOptions" id="total" value="total"
        v-model="currentMode" label="Total-sales" />
      <CFormCheck type="radio" name="radioOptions" id="max" value="max"
        v-model="currentMode" label="Sold receipt" checked />
    </div>
    <div v-else>
      Not connected to services yet
    </div>
  </div>

  <div class="panel info"
    v-if="loadState.storesInfoLoaded && loadState.cachedTotalInitialized && loadState.cachedMaxInitialized">
    <p>ID={{ highestAccountingObj.totalId }} has highest-total-sales: {{
      highestAccountingObj.totalValue }}</p>
    <p>ID={{ highestAccountingObj.maxId }} has highest-max-receipt:
      {{ highestAccountingObj.maxValue }}</p>
  </div>
</template>

<style scoped>
:root {
  font-family: 'Jost', sans-serif;
  font-family: 'Raleway', sans-serif;
  font-weight: 500;
}

#map {
  position: absolute;
  inset: 0;
}

.panel {
  display: flex;
  flex-direction: column;
  justify-content: center;
  margin: 15px;
  padding: 15px;
  border-radius: 22px;
  min-height: 100px;
  background-color: rgba(24, 24, 24, 0.92);
  border: 3px solid #ffffff;
  box-shadow: 0 0 50px -15px rgba(0, 0, 0, 0.899);
  font-family: 'Jost', sans-serif;
  font-family: 'Raleway', sans-serif;
  color: white;
}

.panel.radio {
  position: absolute;
  top: 0;
  left: 0;
}

.panel.info {
  position: absolute;
  top: 0;
  right: 0;
}

p {
  margin-bottom: 5px;
}

.redMarker {
  fill: #d95635
}

.defaultMarker {
  fill: #3fb1ce
}

.popup {
  opacity: 50%;
}
</style>
