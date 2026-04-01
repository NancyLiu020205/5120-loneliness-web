<script setup>
import { computed, onMounted, ref } from 'vue'

const MELBOURNE_CENTER = { lat: -37.8136, lng: 144.9631 }

const mapContainerRef = ref(null)
const query = ref('')
const mapReady = ref(false)
const loadingRooms = ref(false)
const selectedRoomId = ref(null)
const userPosition = ref(null)
const travelMode = ref('WALKING')
const routing = ref(false)

const rooms = ref([])
const routeSummary = ref('')

let map
let userMarker
let directionsService
let directionsRenderer
const roomMarkers = []
const TRAVEL_MODES = [
  { id: 'WALKING', label: 'Walking' },
  { id: 'BICYCLING', label: 'Cycling' },
  { id: 'DRIVING', label: 'Driving' },
  { id: 'TRANSIT', label: 'Transit' },
]

function formatWalkDuration(durationText) {
  if (!durationText) return '--'
  return /walk/i.test(durationText) ? durationText : `${durationText} walk`
}

function parseDistanceToMeters(distanceText) {
  if (!distanceText) return Number.POSITIVE_INFINITY
  const text = distanceText.toLowerCase().replace(/,/g, '').trim()
  if (text.endsWith('km')) return Number.parseFloat(text) * 1000
  if (text.endsWith('m')) return Number.parseFloat(text)
  return Number.POSITIVE_INFINITY
}

function sortRoomsByWalkingDistance() {
  rooms.value = [...rooms.value].sort(
    (a, b) => parseDistanceToMeters(a.distanceText) - parseDistanceToMeters(b.distanceText),
  )
}

function loadGoogleMapsApi() {
  if (window.google?.maps) return Promise.resolve(window.google.maps)

  const apiKey = import.meta.env.VITE_GOOGLE_MAPS_API_KEY
  if (!apiKey) {
    return Promise.reject(
      new Error('Missing VITE_GOOGLE_MAPS_API_KEY. Please configure it in your .env file.'),
    )
  }

  return new Promise((resolve, reject) => {
    const script = document.createElement('script')
    script.src =
      `https://maps.googleapis.com/maps/api/js?key=${apiKey}` +
      '&libraries=places,geometry&v=weekly'
    script.async = true
    script.defer = true
    script.onload = () => resolve(window.google.maps)
    script.onerror = () => reject(new Error('Failed to load Google Maps script.'))
    document.head.appendChild(script)
  })
}

function clearRoomMarkers() {
  roomMarkers.forEach((marker) => marker.setMap(null))
  roomMarkers.length = 0
}

function renderRoomMarkers() {
  if (!map || !window.google?.maps) return

  clearRoomMarkers()
  rooms.value.forEach((room) => {
    const marker = new window.google.maps.Marker({
      map,
      position: room.position,
      title: room.name,
      icon: {
        path: window.google.maps.SymbolPath.CIRCLE,
        scale: 7,
        fillColor: '#ef4444',
        fillOpacity: 1,
        strokeWeight: 2,
        strokeColor: '#ffffff',
      },
    })

    marker.addListener('click', () => selectRoomAndRoute(room))
    roomMarkers.push(marker)
  })
}

function setUserMarker(position) {
  if (!map || !window.google?.maps) return

  userPosition.value = position
  if (userMarker) {
    userMarker.setPosition(position)
  } else {
    userMarker = new window.google.maps.Marker({
      map,
      position,
      title: 'Your location',
      icon: {
        path: window.google.maps.SymbolPath.CIRCLE,
        scale: 8,
        fillColor: '#16a34a',
        fillOpacity: 1,
        strokeWeight: 2,
        strokeColor: '#ffffff',
      },
    })
  }
}

async function fetchRoomsNearby(origin) {
  loadingRooms.value = true
  try {
    const url = new URL('/api/counseling-rooms/nearby', window.location.origin)
    url.searchParams.set('lat', String(origin.lat))
    url.searchParams.set('lng', String(origin.lng))
    url.searchParams.set('city', 'melbourne')
    url.searchParams.set('limit', '20')

    const response = await fetch(url)
    if (!response.ok) throw new Error('API unavailable')

    const payload = await response.json()
    rooms.value = (payload?.data || []).map((item) => ({
      id: item.id,
      name: item.name,
      address: item.address,
      position: { lat: Number(item.latitude), lng: Number(item.longitude) },
      distanceText: item.distanceText || '',
      durationText: item.walkDurationText || '',
    }))
  } catch {
    // Fallback mock data used during frontend integration/testing.
    rooms.value = [
      {
        id: 'm1',
        name: 'Melbourne CBD Wellness Centre',
        address: '120 Collins St, Melbourne VIC 3000',
        position: { lat: -37.8145, lng: 144.9732 },
        distanceText: '500m',
        durationText: '7 min walk',
      },
      {
        id: 'm2',
        name: 'City Psychology Clinic',
        address: '98 Bourke St, Melbourne VIC 3000',
        position: { lat: -37.8127, lng: 144.9671 },
        distanceText: '850m',
        durationText: '12 min walk',
      },
      {
        id: 'm3',
        name: 'Southbank Counseling Hub',
        address: '25 Southbank Blvd, Southbank VIC 3006',
        position: { lat: -37.8222, lng: 144.9648 },
        distanceText: '1.2km',
        durationText: '16 min walk',
      },
    ]
    sortRoomsByWalkingDistance()
  } finally {
    loadingRooms.value = false
    renderRoomMarkers()
  }
}

async function updateDistanceDurationForAll(origin) {
  if (!window.google?.maps || rooms.value.length === 0) return

  const service = new window.google.maps.DistanceMatrixService()
  const destinations = rooms.value.map((room) => room.position)

  const result = await service.getDistanceMatrix({
    origins: [origin],
    destinations,
    travelMode: window.google.maps.TravelMode.WALKING,
    unitSystem: window.google.maps.UnitSystem.METRIC,
  })

  const elements = result?.rows?.[0]?.elements || []
  rooms.value = rooms.value.map((room, index) => {
    const info = elements[index]
    if (!info || info.status !== 'OK') return room
    return {
      ...room,
      distanceText: info.distance?.text || room.distanceText,
      durationText: info.duration?.text || room.durationText,
    }
  })
  sortRoomsByWalkingDistance()
}

async function locateUser() {
  if (!navigator.geolocation) {
    await fetchRoomsNearby(MELBOURNE_CENTER)
    return
  }

  navigator.geolocation.getCurrentPosition(
    async ({ coords }) => {
      const current = { lat: coords.latitude, lng: coords.longitude }
      setUserMarker(current)
      map.panTo(current)
      await fetchRoomsNearby(current)
      await updateDistanceDurationForAll(current)
    },
    async () => {
      setUserMarker(MELBOURNE_CENTER)
      await fetchRoomsNearby(MELBOURNE_CENTER)
      await updateDistanceDurationForAll(MELBOURNE_CENTER)
    },
    { enableHighAccuracy: true, timeout: 8000 },
  )
}

async function drawRoute(origin, destination, mode = travelMode.value) {
  if (!directionsService || !directionsRenderer || !window.google?.maps) return

  const request = {
    origin,
    destination,
    travelMode: window.google.maps.TravelMode[mode],
  }
  if (mode === 'TRANSIT') request.transitOptions = { departureTime: new Date() }

  const result = await directionsService.route(request)

  directionsRenderer.setDirections(result)
  const leg = result?.routes?.[0]?.legs?.[0]
  routeSummary.value = leg ? `${leg.distance?.text || ''} | ${leg.duration?.text || ''}` : ''
}

async function selectRoomAndRoute(room) {
  selectedRoomId.value = room.id

  const origin = userPosition.value || MELBOURNE_CENTER
  routing.value = true
  try {
    await drawRoute(origin, room.position)
  } finally {
    routing.value = false
  }
}

async function generateSelectedRoute() {
  const selectedRoom = rooms.value.find((room) => room.id === selectedRoomId.value)
  if (!selectedRoom) return

  const origin = userPosition.value || MELBOURNE_CENTER
  routing.value = true
  try {
    await drawRoute(origin, selectedRoom.position, travelMode.value)
  } finally {
    routing.value = false
  }
}

function clearSelectedRoom() {
  selectedRoomId.value = null
}

async function selectTravelMode(modeId) {
  travelMode.value = modeId
  await generateSelectedRoute()
}

function getCurrentLocationLabel() {
  return userPosition.value ? 'Current location' : 'Melbourne CBD'
}

const filteredRooms = computed(() => {
  const keyword = query.value.trim().toLowerCase()
  if (!keyword) return rooms.value
  return rooms.value.filter((room) => room.name.toLowerCase().includes(keyword))
})

const selectedRoom = computed(() => rooms.value.find((room) => room.id === selectedRoomId.value) || null)

const displayedRooms = computed(() => {
  if (selectedRoom.value) return [selectedRoom.value]
  return filteredRooms.value
})

onMounted(async () => {
  await loadGoogleMapsApi()

  map = new window.google.maps.Map(mapContainerRef.value, {
    center: MELBOURNE_CENTER,
    zoom: 13,
    mapTypeControl: false,
    streetViewControl: false,
    fullscreenControl: false,
  })

  directionsService = new window.google.maps.DirectionsService()
  directionsRenderer = new window.google.maps.DirectionsRenderer({
    map,
    suppressMarkers: false,
    polylineOptions: { strokeColor: '#059669', strokeWeight: 5 },
  })

  mapReady.value = true
  await locateUser()
})
</script>

<template>
  <main class="page">
    <section class="top-bar">
      <div class="search-wrapper">
        <input
          v-model="query"
          class="search-input"
          type="text"
          placeholder="Find nearby counseling rooms"
        />
      </div>
      <button class="location-btn" type="button" @click="locateUser">Use My Location</button>
    </section>

    <section class="layout">
      <div class="map-panel">
        <div ref="mapContainerRef" class="map-canvas"></div>
        <div v-if="!mapReady" class="map-mask">Loading map...</div>
        <div class="legend">
          <div class="legend-title">Legend</div>
          <div class="legend-item"><span class="dot user-dot"></span>Your Location</div>
          <div class="legend-item"><span class="dot room-dot"></span>Nearby Counseling Rooms</div>
        </div>
      </div>

      <aside class="list-panel">
        <h2>Nearby Counseling Rooms</h2>
        <p class="sub">Based on your current location</p>
        <button v-if="selectedRoom" type="button" class="back-btn" @click="clearSelectedRoom">
          Back to full list
        </button>

        <div v-if="loadingRooms" class="state-tip">Loading nearby rooms...</div>
        <div v-else-if="displayedRooms.length === 0" class="state-tip">
          No counseling rooms found.
        </div>

        <button
          v-for="room in displayedRooms"
          :key="room.id"
          type="button"
          :class="['room-card', { active: selectedRoomId === room.id }]"
          @click="selectRoomAndRoute(room)"
        >
          <h3>{{ room.name }}</h3>
          <p class="meta">{{ room.distanceText || '--' }} | {{ formatWalkDuration(room.durationText) }}</p>
          <p class="origin-line">From: {{ getCurrentLocationLabel() }}</p>
          <span class="details-btn">View Routes</span>
        </button>

        <section v-if="selectedRoom" class="route-builder">
          <h3 class="route-title">Travel Mode</h3>
          <div class="mode-row">
            <button
              v-for="mode in TRAVEL_MODES"
              :key="mode.id"
              type="button"
              :class="['mode-chip', { active: travelMode === mode.id }]"
              @click="selectTravelMode(mode.id)"
            >
              {{ mode.label }}
            </button>
          </div>
          <p v-if="routeSummary" class="estimate-text">Estimate: {{ routeSummary }}</p>
        </section>
      </aside>
    </section>
  </main>
</template>

<style scoped>
.page {
  max-width: 1400px;
  margin: 0 auto;
  padding: 18px 24px 24px;
}

.top-bar {
  display: grid;
  grid-template-columns: 1fr 150px;
  gap: 12px;
  margin-bottom: 14px;
}

.search-input {
  width: 100%;
  border: 1px solid #d1d5db;
  border-radius: 10px;
  background: #ffffff;
  padding: 11px 12px;
  font-size: 14px;
  outline: none;
}

.search-input:focus {
  border-color: #059669;
  box-shadow: 0 0 0 3px #d1fae5;
}

.location-btn {
  border: none;
  border-radius: 10px;
  background: #16a34a;
  color: #ffffff;
  font-weight: 600;
  cursor: pointer;
}

.layout {
  display: grid;
  grid-template-columns: 2.2fr 1fr;
  gap: 14px;
  min-height: 72vh;
}

.map-panel {
  position: relative;
  background: #dfe8d9;
  border: 1px solid #d1d5db;
  border-radius: 12px;
  overflow: hidden;
}

.map-canvas {
  width: 100%;
  height: 100%;
  min-height: 72vh;
}

.map-mask {
  position: absolute;
  inset: 0;
  display: grid;
  place-items: center;
  background: rgba(255, 255, 255, 0.8);
  color: #374151;
}

.legend {
  position: absolute;
  left: 12px;
  bottom: 12px;
  background: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 10px;
  min-width: 160px;
}

.legend-title {
  font-size: 12px;
  font-weight: 700;
  margin-bottom: 6px;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 12px;
  color: #4b5563;
}

.dot {
  width: 10px;
  height: 10px;
  border-radius: 999px;
}

.user-dot {
  background: #16a34a;
}

.room-dot {
  background: #ef4444;
}

.list-panel {
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  background: #f8fafc;
  padding: 14px;
  overflow: auto;
}

h2 {
  margin: 0;
  color: #1f2937;
}

.sub {
  margin: 4px 0 12px;
  color: #6b7280;
  font-size: 13px;
}

.back-btn {
  margin-bottom: 12px;
  border: 1px solid #cbd5e1;
  background: #ffffff;
  color: #334155;
  font-size: 13px;
  font-weight: 700;
  border-radius: 8px;
  padding: 8px 10px;
  cursor: pointer;
}

.route-summary {
  margin: 0 0 12px;
  color: #065f46;
  font-size: 13px;
  font-weight: 600;
}

.state-tip {
  border: 1px dashed #d1d5db;
  border-radius: 10px;
  padding: 12px;
  color: #6b7280;
}

.room-card {
  width: 100%;
  text-align: left;
  border: 1px solid #d1d5db;
  border-radius: 12px;
  background: #ffffff;
  padding: 12px;
  margin-bottom: 12px;
  cursor: pointer;
}

.room-card.active {
  border-color: #059669;
  box-shadow: 0 0 0 2px #d1fae5;
}

h3 {
  margin: 0 0 10px;
  color: #1f2937;
  font-size: 19px;
}

.meta {
  margin: 0 0 8px;
  color: #374151;
}

.origin-line {
  margin: 0 0 12px;
  color: #6b7280;
  font-size: 13px;
}

.details-btn {
  display: inline-block;
  width: 100%;
  text-align: center;
  border: 1px solid #059669;
  border-radius: 8px;
  color: #065f46;
  font-weight: 600;
  line-height: 34px;
}

.route-builder {
  margin-top: 8px;
  border-top: 1px solid #e5e7eb;
  padding-top: 14px;
}

.route-title {
  margin: 0 0 10px;
  font-size: 16px;
  color: #1f2937;
}

.mode-row {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.mode-chip {
  border: 1px solid #cbd5e1;
  border-radius: 999px;
  background: #ffffff;
  padding: 10px 16px;
  color: #334155;
  font-size: 15px;
  font-weight: 700;
  cursor: pointer;
}

.mode-chip.active {
  background: #16a34a;
  border-color: #16a34a;
  color: #ffffff;
}

.estimate-text {
  margin: 10px 0 0;
  font-size: 16px;
  font-weight: 700;
  color: #166534;
}

@media (max-width: 1200px) {
  .layout {
    grid-template-columns: 1fr;
  }

  .map-canvas {
    min-height: 58vh;
  }
}

@media (max-width: 900px) {
  .top-bar {
    grid-template-columns: 1fr;
  }
}
</style>
