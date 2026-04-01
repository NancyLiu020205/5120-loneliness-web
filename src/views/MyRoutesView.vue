<script setup>
import { computed, onMounted, ref } from 'vue'

const mapContainerRef = ref(null)
const mapReady = ref(false)

const startLocation = ref('')
const destination = ref('')
const activePreference = ref('socially-active') // 'socially-active' or 'nature-shade'

let map
let infoWindow

// Dummy features matching the UI requested
const mockFacilities = [
  {
    type: 'toilet',
    name: 'Swanston St Public Toilet',
    desc: 'Public Toilet | Open 24/7',
    tag: 'Accessible',
    position: { lat: -37.8115, lng: 144.9650 }
  },
  {
    type: 'bench',
    name: 'Rest Bench',
    desc: 'Rest Bench | Shaded Seating',
    tag: 'Shaded',
    position: { lat: -37.8095, lng: 144.9680 } // slightly modified position to place the bench near the end
  }
]

// Mock points for a route line from logic
const mockStart = { lat: -37.8136, lng: 144.9600 }
const mockEnd = { lat: -37.8080, lng: 144.9700 }

function loadGoogleMapsApi() {
  if (window.google?.maps) return Promise.resolve(window.google.maps)

  const apiKey = import.meta.env.VITE_GOOGLE_MAPS_API_KEY
  if (!apiKey) {
    return Promise.reject(new Error('Missing VITE_GOOGLE_MAPS_API_KEY'))
  }

  return new Promise((resolve, reject) => {
    const script = document.createElement('script')
    script.src = `https://maps.googleapis.com/maps/api/js?key=${apiKey}&libraries=places,geometry&v=weekly`
    script.async = true
    script.defer = true
    script.onload = () => resolve(window.google.maps)
    script.onerror = () => reject(new Error('Google Maps script load failed'))
    document.head.appendChild(script)
  })
}

function initMockMap() {
  map = new window.google.maps.Map(mapContainerRef.value, {
    center: { lat: -37.8105, lng: 144.9660 }, // Center map optimally around route
    zoom: 15,
    mapTypeControl: false,
    streetViewControl: false,
    fullscreenControl: false,
    styles: [
      {
        featureType: "landscape",
        elementType: "geometry",
        stylers: [{ color: "#eef3eb" }] // soft green tone from design
      },
      {
        featureType: "poi.park",
        elementType: "geometry",
        stylers: [{ color: "#dff1d3" }]
      }
    ]
  })

  infoWindow = new window.google.maps.InfoWindow()

  // 1. Draw Route line
  new window.google.maps.Polyline({
    path: [mockStart, mockEnd],
    geodesic: true,
    strokeColor: '#2b8a3e', 
    strokeOpacity: 1.0,
    strokeWeight: 4,
    map: map,
  })

  // 2. Add Start & Destination markers matching UI
  new window.google.maps.Marker({
    position: mockStart,
    map,
    icon: {
      path: window.google.maps.SymbolPath.CIRCLE,
      scale: 7,
      fillColor: '#2b8a3e', // Green start dot
      fillOpacity: 1,
      strokeColor: '#ffffff',
      strokeWeight: 2,
    },
    label: { text: 'Start', color: '#111', fontSize: '11px', fontWeight: 'bold', className: 'marker-lbl-bottom' }
  })

  new window.google.maps.Marker({
    position: mockEnd,
    map,
    icon: {
      path: window.google.maps.SymbolPath.CIRCLE,
      scale: 8,
      fillColor: '#dc2626', // Red end dot
      fillOpacity: 1,
      strokeColor: '#ffffff',
      strokeWeight: 2,
    },
    label: { text: 'Destination', color: '#111', fontSize: '11px', fontWeight: 'bold', className: 'marker-lbl-top' }
  })

  // 3. Add Custom Facility Markers (Toilets, Benches)
  mockFacilities.forEach(facility => {
    const isToilet = facility.type === 'toilet'
    const color = isToilet ? '#3b82f6' : '#f59e0b'
    const labelText = isToilet ? 'WC' : ''
    
    // Rounded square SVG path
    const svgMarker = {
      path: 'M 4,0 H 20 C 22.2,0 24,1.8 24,4 V 20 C 24,22.2 22.2,24 20,24 H 4 C 1.8,24 0,22.2 0,20 V 4 C 0,1.8 1.8,0 4,0 Z',
      fillColor: color,
      fillOpacity: 1,
      strokeWeight: 0,
      scale: 1,
      anchor: new window.google.maps.Point(12, 12),
      labelOrigin: new window.google.maps.Point(12, 12)
    }

    const marker = new window.google.maps.Marker({
      position: facility.position,
      map: map,
      icon: svgMarker,
      label: {
        text: labelText,
        color: '#ffffff',
        fontSize: '10px',
        fontWeight: 'bold'
      }
    })

    // 4. Info Window Tap-to-preview overlay
    const contentStr = `
      <div style="padding: 10px 14px; font-family: Inter, sans-serif; min-width: 200px; text-align: center;">
        <strong style="display:block; font-size: 15px; margin-bottom: 6px; color:#1f2937;">${facility.name}</strong>
        <span style="display:block; font-size: 13px; color:#4b5563; margin-bottom: 8px;">${facility.desc}</span>
        <span style="display:inline-block; font-size: 12px; padding: 4px 8px; background:#dcfce7; color:#166534; border-radius:4px; font-weight:600;">${facility.tag}</span>
      </div>
    `

    marker.addListener('click', () => {
      infoWindow.setContent(contentStr)
      infoWindow.open({
        anchor: marker,
        map,
      })
    })

    // To mimic the screenshot, optionally auto-open the toilet's infowindow
    if (isToilet) {
      setTimeout(() => {
        infoWindow.setContent(contentStr)
        infoWindow.open({ anchor: marker, map })
      }, 500)
    }
  })
}

onMounted(async () => {
  try {
    await loadGoogleMapsApi()
    initMockMap()
    mapReady.value = true
  } catch (err) {
    console.error("Map initialization error:", err)
  }
})

function useMyLocation() {
  startLocation.value = "Current Location"
}

function generateRoute() {
  console.log('Generating route with preference:', activePreference.value)
}

function setPreference(pref) {
  activePreference.value = pref
}
</script>

<template>
  <div class="my-routes-page">
    <aside class="sidebar">
      <h1 class="page-title">Plan Your Route</h1>

      <div class="form-group">
        <label class="form-label label-green">A Your Location</label>
        <div class="input-row">
          <div class="input-icon-wrapper">
            <span class="icon-magnify">🔍</span>
            <input v-model="startLocation" type="text" placeholder="Enter your starting location..." />
          </div>
          <button class="btn-sm btn-green" @click="useMyLocation">Use My Location</button>
        </div>
      </div>

      <div class="form-group">
        <label class="form-label label-red">B Destination</label>
        <div class="input-row">
          <div class="input-icon-wrapper">
             <span class="icon-magnify">🔍</span>
             <input v-model="destination" type="text" placeholder="Enter your destination..." />
          </div>
        </div>
      </div>

      <button class="btn-generate" @click="generateRoute">Generate Route</button>

      <div class="preferences-section">
        <h3 class="pref-title">Route Preferences</h3>
        
        <button 
          :class="['pref-btn', activePreference === 'socially-active' ? 'active' : '']"
          @click="setPreference('socially-active')"
        >
          <span class="pref-btn-title">Socially Active</span>
          <span class="pref-btn-desc">Routes with high pedestrian density</span>
        </button>

        <button 
          :class="['pref-btn', activePreference === 'nature-shade' ? 'active' : '']"
          @click="setPreference('nature-shade')"
        >
          <span class="pref-btn-title">Nature & Shade</span>
          <span class="pref-btn-desc">Tree canopy coverage >= 60%</span>
        </button>
      </div>
    </aside>

    <main class="map-container">
      <div ref="mapContainerRef" class="map-view"></div>
      
      <div v-if="!mapReady" class="map-loading-mask">Loading map layout...</div>

      <!-- Features Overlaid on Map -->
      <button class="map-view-toggle">Map View</button>

      <div class="legend-box">
        <h4>Legend</h4>
        <div class="legend-item"><span class="legend-color toilet-color"></span> Public Toilet</div>
        <div class="legend-item"><span class="legend-color bench-color"></span> Rest Bench</div>
        <div class="legend-item"><span class="legend-color route-color"></span> Walking Route</div>
      </div>

      <div class="gap-alert-box">
        Warning: 600m stretch without rest facilities ahead. Please plan your pace accordingly.
      </div>
    </main>
  </div>
</template>

<style scoped>
.my-routes-page {
  display: flex;
  min-height: calc(100vh - 64px);
  background: #f8fafc;
  font-family: 'Inter', system-ui, -apple-system, sans-serif;
}

.sidebar {
  width: 420px;
  background: #f8fafc;
  padding: 32px;
  border-right: 1px solid #e2e8f0;
  display: flex;
  flex-direction: column;
}

.page-title {
  font-size: 26px;
  font-weight: 500;
  color: #1e293b;
  margin: 0 0 32px 0;
}

.form-group {
  margin-bottom: 24px;
}

.form-label {
  display: block;
  font-size: 14px;
  font-weight: 700;
  margin-bottom: 8px;
}
.label-green { color: #16a34a; }
.label-red { color: #dc2626; }

.input-row {
  display: flex;
  gap: 12px;
  align-items: center;
}

.input-icon-wrapper {
  position: relative;
  flex: 1;
}

.input-icon-wrapper input {
  box-sizing: border-box; /* Fix for overlapping: prevents width 100% + padding from exceeding container */
  width: 100%;
  padding: 12px 14px 12px 36px;
  border: 1px solid #cbd5e1;
  border-radius: 10px;
  font-size: 15px;
  outline: none;
  background: #fff;
  transition: all 0.2s;
}

.input-icon-wrapper input:focus {
  border-color: #16a34a;
  box-shadow: 0 0 0 3px #dcfce7;
}

.icon-magnify {
  position: absolute;
  left: 12px;
  top: 50%;
  transform: translateY(-50%);
  font-size: 16px;
  color: #94a3b8;
}

.btn-sm {
  padding: 12px 16px;
  font-size: 14px;
  font-weight: 700;
  border-radius: 10px;
  border: none;
  cursor: pointer;
  white-space: nowrap;
  transition: background 0.2s;
  flex-shrink: 0; /* Ensures the button never shrinks inside the flex row */
}

.btn-green {
  background: #16a34a;
  color: white;
}
.btn-green:hover { background: #15803d; }

.btn-generate {
  width: 160px;
  padding: 14px;
  background: #16a34a;
  color: white;
  font-weight: 700;
  font-size: 16px;
  border-radius: 10px;
  border: none;
  cursor: pointer;
  margin-top: 8px;
  margin-bottom: 40px;
  transition: background 0.2s;
}
.btn-generate:hover { background: #15803d; }

.preferences-section {
  margin-top: auto;
}

.pref-title {
  font-size: 18px;
  font-weight: 700;
  color: #1e293b;
  margin-bottom: 16px;
}

.pref-btn {
  display: block;
  width: 100%;
  text-align: center;
  padding: 16px;
  border-radius: 12px;
  border: none;
  margin-bottom: 14px;
  cursor: pointer;
  transition: all 0.2s;
  background: #94a3b8; /* Inactive grey */
  color: white;
  opacity: 0.9;
}

.pref-btn.active {
  background: #64748b; /* Active slightly darker slate/grey */
  opacity: 1;
  transform: translateY(-1px);
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
}

.pref-btn-title {
  display: block;
  font-weight: 700;
  font-size: 16px;
  margin-bottom: 4px;
}

.pref-btn-desc {
  display: block;
  font-size: 13px;
  opacity: 0.9;
}


.map-container {
  flex: 1;
  position: relative;
  margin: 16px 16px 16px 0;
  border-radius: 12px;
  overflow: hidden;
  border: 1px solid #cbd5e1;
  background: #eef3eb; 
}

.map-view {
  width: 100%;
  height: 100%;
  border-radius: 12px;
}

.map-loading-mask {
  position: absolute;
  inset: 0;
  background: rgba(255, 255, 255, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  color: #475569;
  z-index: 5;
}

.map-view-toggle {
  position: absolute;
  top: 16px;
  right: 16px;
  background: white;
  border: 1px solid #cbd5e1;
  padding: 8px 16px;
  font-size: 14px;
  font-weight: 700;
  border-radius: 8px;
  cursor: pointer;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
  color: #1e293b;
  z-index: 10;
}

.legend-box {
  position: absolute;
  left: 24px;
  bottom: 84px;
  background: white;
  padding: 14px 18px;
  border-radius: 10px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.08);
  font-size: 13px;
  color: #475569;
  z-index: 10;
  border: 1px solid #e2e8f0;
}

.legend-box h4 {
  margin: 0 0 10px 0;
  font-weight: 700;
  color: #1e293b;
  font-size: 14px;
}

.legend-item {
  display: flex;
  align-items: center;
  margin-bottom: 8px;
}
.legend-item:last-child {
  margin-bottom: 0;
}

.legend-color {
  display: inline-block;
  width: 14px;
  height: 14px;
  border-radius: 4px;
  margin-right: 10px;
}
.toilet-color { background: #3b82f6; }
.bench-color { background: #f59e0b; }
.route-color { background: #16a34a; height: 4px; border-radius: 2px; }

.gap-alert-box {
  position: absolute;
  bottom: 24px;
  left: 50%;
  transform: translateX(-50%);
  background: #fef3c7;
  color: #854d0e;
  border: 1px solid #fcd34d;
  padding: 12px 28px;
  border-radius: 10px;
  font-size: 14px;
  font-weight: 600;
  box-shadow: 0 4px 12px rgba(0,0,0,0.08);
  z-index: 10;
  white-space: nowrap;
}

@media (max-width: 1024px) {
  .my-routes-page {
    flex-direction: column;
    min-height: auto;
  }
  .sidebar {
    width: 100%;
    border-right: none;
    border-bottom: 1px solid #e2e8f0;
    padding: 24px;
  }
  .map-container {
    height: 65vh;
    margin: 16px;
  }
}
</style>
