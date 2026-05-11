<script setup>
import { computed, nextTick, onMounted, onUnmounted, ref, watch } from 'vue'
import { useRouter } from 'vue-router'

const router = useRouter()

const STORAGE_KEY_CATEGORIES = 'discoverPlaces.selectedCategories'
const STORAGE_KEY_RADIUS = 'discoverPlaces.selectedRadius'
const PLACES_PER_PAGE = 20
const MAX_MAP_MARKERS = 300

const CATEGORY_OPTIONS = [
  { key: 'landmarks', label: 'Landmarks', tagLabel: 'Landmark', icon: '🏛️' },
  {
    key: 'cafes_restaurants',
    label: 'Cafes & restaurants',
    tagLabel: 'Cafe/Restaurant',
    icon: '☕',
  },
  { key: 'artworks_fountains', label: 'Artworks & fountains', tagLabel: 'Artwork', icon: '🎨' },
  {
    key: 'memorials_sculptures',
    label: 'Memorials & sculptures',
    tagLabel: 'Memorial',
    icon: '🗿',
  },
]

const RADIUS_OPTIONS = [
  { meters: 500, label: '500 m' },
  { meters: 1000, label: '1 km' },
  { meters: 2000, label: '2 km' },
  { meters: 3000, label: 'Exceed 2 km' },
]

const DEFAULT_CATEGORY_KEYS = CATEGORY_OPTIONS.map((option) => option.key)
const DEFAULT_RADIUS = 1000
const DISCOVER_API_BASE_URL = 'https://mk3ban19bb.execute-api.ap-southeast-2.amazonaws.com'
const DISCOVER_DEFAULT_LIMIT = 200
const VENUES_DEFAULT_LIMIT = 5000
const VENUES_EXTENDED_LIMIT = 15000
const EXCEED_2KM_QUERY_RADIUS = 20000
const EXCEED_2KM_QUERY_LIMIT = 1000

const categoryMetaByKey = CATEGORY_OPTIONS.reduce((acc, option) => {
  acc[option.key] = option
  return acc
}, {})
const mapMarkerColorByCategory = {
  landmarks: '#f97316',
  cafes_restaurants: '#ec4899',
  artworks_fountains: '#22c55e',
  memorials_sculptures: '#3b82f6',
}

const allPlaces = ref([])
const isLoadingPlaces = ref(true)
const loadError = ref('')
const locationUnavailable = ref(false)
const selectedCategories = ref([...DEFAULT_CATEGORY_KEYS])
const selectedRadius = ref(DEFAULT_RADIUS)
const currentPage = ref(1)
const userLocation = ref(null)
const addressQuery = ref('')
const addressInputRef = ref(null)
const mapContainerRef = ref(null)
const applyingAddressFilter = ref(false)
const addressFilterError = ref('')
const mapError = ref('')
const locationMode = ref('none') // none | device | address
const activeDetailPlace = ref(null)
const detailPanelState = ref('closed') // closed | opening | open | closing
const pendingDetailPlace = ref(null)
const directionsError = ref('')
const isIdeasModalOpen = ref(false)
const ideasStep = ref(1)
const ideasTransportMode = ref('')
const ideasCategoryAnswers = ref([])
const detailById = ref(new Map())
const loadingDetailIds = ref(new Set())
const venuesCacheByQuery = new Map()

const IDEAS_CATEGORY_CHOICES = [
  {
    key: 'history_story',
    label: 'I want history stories and iconic city sights',
    categoryKey: 'landmarks',
  },
  {
    key: 'creative_relax',
    label: 'I want creative vibes, artworks, and waterside views',
    categoryKey: 'artworks_fountains',
  },
  {
    key: 'quiet_reflect',
    label: 'I want a quiet, reflective walk with memorial spaces',
    categoryKey: 'memorials_sculptures',
  },
]

function pickFirstDefined(...values) {
  for (const value of values) {
    if (value !== undefined && value !== null) return value
  }
  return null
}

function toFiniteNumber(value) {
  if (value === undefined || value === null || value === '') return null
  const numeric = Number(value)
  return Number.isFinite(numeric) ? numeric : null
}

function normalizeApiPayload(payload) {
  if (!payload || typeof payload !== 'object') return payload

  // Support API Gateway/Lambda proxy responses: { statusCode, body: "..." }.
  if (typeof payload.body === 'string') {
    try {
      return JSON.parse(payload.body)
    } catch {
      return payload
    }
  }
  return payload
}

function loadGoogleMapsApi() {
  if (window.google?.maps) return Promise.resolve(window.google.maps)

  const apiKey = import.meta.env.VITE_GOOGLE_MAPS_API_KEY
  if (!apiKey) {
    return Promise.reject(new Error('Missing VITE_GOOGLE_MAPS_API_KEY.'))
  }

  return new Promise((resolve, reject) => {
    const script = document.createElement('script')
    script.src =
      `https://maps.googleapis.com/maps/api/js?key=${apiKey}` + '&libraries=places&v=weekly'
    script.async = true
    script.defer = true
    script.onload = () => resolve(window.google.maps)
    script.onerror = () => reject(new Error('Failed to load Google Maps script.'))
    document.head.appendChild(script)
  })
}

function normalizeCategory(rawCategory) {
  const text = String(rawCategory || '')
    .trim()
    .toLowerCase()
    .replace(/[_-]+/g, ' ')
  if (text.includes('cafe') || text.includes('restaurant') || text.includes('dining'))
    return 'cafes_restaurants'
  if (text.includes('landmark')) return 'landmarks'
  if (text.includes('art') || text.includes('fountain')) return 'artworks_fountains'
  if (text.includes('memorial') || text.includes('sculpture') || text.includes('monument'))
    return 'memorials_sculptures'
  return 'landmarks'
}

function normalizePlace(input, index, sourceType = 'places') {
  const categoryRaw = pickFirstDefined(
    input.category,
    input.type,
    input.place_category,
    input.category_name,
    input.categoryName,
  )
  const categoryKey =
    normalizeCategory(categoryRaw) || (sourceType === 'venues' ? 'cafes_restaurants' : 'landmarks')
  if (!categoryKey) return null

  const latCandidate = toFiniteNumber(
    pickFirstDefined(input.latitude, input.lat, input.location?.lat, input.location_lat, input.y),
  )
  const lngCandidate = toFiniteNumber(
    pickFirstDefined(input.longitude, input.lng, input.location?.lng, input.location_lng, input.x),
  )
  const distanceMetersCandidate = toFiniteNumber(
    pickFirstDefined(
      input.distance_meters,
      input.distanceMeters,
      input.distance_meter,
      input.approx_distance_m,
      input.distance,
    ),
  )
  const hasCoords = latCandidate !== null && lngCandidate !== null
  const name = String(
    pickFirstDefined(input.name, input.place_name, input.placeName, 'Unnamed place'),
  )
  const address = String(
    pickFirstDefined(
      input.address,
      input.formatted_address,
      input.location_address,
      input.locationAddress,
      `${name}, Melbourne VIC`,
    ) || '',
  )
  const isRichDetailCategory = categoryKey !== 'landmarks'
  const artistOrSubjectRaw = pickFirstDefined(
    input.artist,
    input.artist_name,
    input.subject,
    input.artist_or_subject,
  )
  const yearRaw = pickFirstDefined(input.year, input.year_created, input.period)
  const materialRaw = pickFirstDefined(input.material, input.materials, input.medium)
  const descriptionRaw = pickFirstDefined(input.description, input.summary, input.details)
  const workTitleRaw = pickFirstDefined(input.work_title, input.workTitle, input.title, name)
  const resolvedDescription = (() => {
    if (descriptionRaw !== undefined && descriptionRaw !== null && String(descriptionRaw).trim()) {
      return String(descriptionRaw)
    }
    if (isRichDetailCategory) {
      return `${name} is a notable Melbourne public work that contributes to the city's cultural streetscape.`
    }
    return ''
  })()

  return {
    id: `${sourceType}-${String(
      pickFirstDefined(input.id, input.place_id, input.placeId, input.uuid, `place-${index}`),
    )}`,
    sourceId: String(
      pickFirstDefined(input.id, input.place_id, input.placeId, input.uuid, `place-${index}`),
    ),
    sourceType,
    name,
    categoryKey,
    categoryLabel: categoryMetaByKey[categoryKey].label,
    lat: hasCoords ? latCandidate : null,
    lng: hasCoords ? lngCandidate : null,
    address,
    distanceMetersFromApi: distanceMetersCandidate,
    imageUrl: String(
      pickFirstDefined(input.image_url, input.imageUrl, input.image, input.thumbnail_url, '') || '',
    ),
    icon: String(input.icon ?? '').trim() || categoryMetaByKey[categoryKey].icon,
    infoUrl: String(
      pickFirstDefined(
        input.more_info_url,
        input.moreInfoUrl,
        input.more_info_link,
        input.detail_url,
        input.website,
        input.url,
        '',
      ) || '',
    ),
    artistOrSubject: isRichDetailCategory
      ? String(artistOrSubjectRaw || 'City of Melbourne collection')
      : '',
    year: isRichDetailCategory ? String(yearRaw || 'Unknown') : '',
    workTitle: isRichDetailCategory ? String(workTitleRaw || name) : '',
    material: isRichDetailCategory ? String(materialRaw || 'Mixed materials') : '',
    description: resolvedDescription,
  }
}

function toRadians(value) {
  return (value * Math.PI) / 180
}

function calculateDistanceMeters(from, to) {
  const earthRadiusMeters = 6371000
  const dLat = toRadians(to.lat - from.lat)
  const dLng = toRadians(to.lng - from.lng)
  const lat1 = toRadians(from.lat)
  const lat2 = toRadians(to.lat)

  const a =
    Math.sin(dLat / 2) * Math.sin(dLat / 2) +
    Math.cos(lat1) * Math.cos(lat2) * Math.sin(dLng / 2) * Math.sin(dLng / 2)
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a))
  return earthRadiusMeters * c
}

function formatDistance(distanceMeters) {
  if (distanceMeters < 1000) return `Approx. ${Math.round(distanceMeters)} m`
  return `Approx. ${(distanceMeters / 1000).toFixed(1)} km`
}

function isPlaceWithinSelectedRadiusBand(distanceMeters) {
  if (typeof distanceMeters !== 'number') return false
  if (selectedRadius.value === 500) return distanceMeters <= 500
  if (selectedRadius.value === 1000) return distanceMeters > 500 && distanceMeters <= 1000
  if (selectedRadius.value === 2000) return distanceMeters > 1000 && distanceMeters <= 2000
  if (selectedRadius.value === 3000) return distanceMeters > 2000
  return false
}

function parsePlacesPayload(payload) {
  const normalized = normalizeApiPayload(payload)
  if (Array.isArray(normalized)) return normalized
  if (Array.isArray(normalized?.data)) return normalized.data
  if (Array.isArray(normalized?.places)) return normalized.places
  if (Array.isArray(normalized?.venues)) return normalized.venues
  if (Array.isArray(normalized?.items)) return normalized.items
  if (Array.isArray(normalized?.results)) return normalized.results
  if (Array.isArray(normalized?.records)) return normalized.records
  if (normalized && typeof normalized === 'object') return [normalized]
  return []
}

function buildQueryOptions() {
  const queryLimit = selectedRadius.value === 3000 ? EXCEED_2KM_QUERY_LIMIT : DISCOVER_DEFAULT_LIMIT
  const queryRadius = selectedRadius.value === 3000 ? EXCEED_2KM_QUERY_RADIUS : selectedRadius.value
  return { queryLimit, queryRadius }
}

function buildPlacesApiUrl() {
  const configuredBase = String(import.meta.env.VITE_DISCOVER_PLACES_API_BASE_URL || '').trim()
  const baseUrl = configuredBase || DISCOVER_API_BASE_URL
  const endpoint = new URL('/places', baseUrl.endsWith('/') ? baseUrl : `${baseUrl}/`)
  const { queryLimit, queryRadius } = buildQueryOptions()
  endpoint.searchParams.set('limit', String(queryLimit))
  endpoint.searchParams.set('radius', String(queryRadius))
  if (userLocation.value) {
    endpoint.searchParams.set('lat', String(userLocation.value.lat))
    endpoint.searchParams.set('lng', String(userLocation.value.lng))
  }
  return endpoint.toString()
}

function buildVenuesApiUrl() {
  const configuredBase = String(import.meta.env.VITE_DISCOVER_PLACES_API_BASE_URL || '').trim()
  const baseUrl = configuredBase || DISCOVER_API_BASE_URL
  const endpoint = new URL('/venues', baseUrl.endsWith('/') ? baseUrl : `${baseUrl}/`)
  const { queryRadius } = buildQueryOptions()
  // Venues around CBD are dense; wider radius bands need a higher limit to avoid top-N truncation.
  const venuesLimit =
    selectedRadius.value === 2000 || selectedRadius.value === 3000
      ? VENUES_EXTENDED_LIMIT
      : VENUES_DEFAULT_LIMIT
  endpoint.searchParams.set('limit', String(venuesLimit))
  endpoint.searchParams.set('radius', String(queryRadius))
  if (userLocation.value) {
    endpoint.searchParams.set('lat', String(userLocation.value.lat))
    endpoint.searchParams.set('lng', String(userLocation.value.lng))
  }
  return endpoint.toString()
}

function buildVenuesCacheKey(queryRadius) {
  const lat = userLocation.value?.lat
  const lng = userLocation.value?.lng
  const latPart = Number.isFinite(lat) ? lat.toFixed(5) : 'default-lat'
  const lngPart = Number.isFinite(lng) ? lng.toFixed(5) : 'default-lng'
  return `${latPart}|${lngPart}|${queryRadius}`
}

function readSessionState() {
  try {
    const storedCategories = JSON.parse(sessionStorage.getItem(STORAGE_KEY_CATEGORIES) || '[]')
    if (Array.isArray(storedCategories)) {
      const valid = storedCategories.filter((key) => DEFAULT_CATEGORY_KEYS.includes(key))
      if (valid.length > 0 || storedCategories.length === 0) selectedCategories.value = valid
    }
    const storedRadius = Number(sessionStorage.getItem(STORAGE_KEY_RADIUS))
    if (RADIUS_OPTIONS.some((option) => option.meters === storedRadius)) {
      selectedRadius.value = storedRadius
    }
  } catch {
    selectedCategories.value = [...DEFAULT_CATEGORY_KEYS]
    selectedRadius.value = DEFAULT_RADIUS
  }
}

function persistSessionState() {
  sessionStorage.setItem(STORAGE_KEY_CATEGORIES, JSON.stringify(selectedCategories.value))
  sessionStorage.setItem(STORAGE_KEY_RADIUS, String(selectedRadius.value))
}

watch([selectedCategories, selectedRadius], () => persistSessionState(), { deep: true })

const placesWithDistance = computed(() => {
  if (!userLocation.value) {
    return allPlaces.value.map((place) => ({
      ...place,
      distanceMeters: toFiniteNumber(place.distanceMetersFromApi),
    }))
  }
  return allPlaces.value.map((place) => {
    const distanceFromApi = toFiniteNumber(place.distanceMetersFromApi)
    if (distanceFromApi !== null) {
      return { ...place, distanceMeters: distanceFromApi }
    }
    if (Number.isFinite(place.lat) && Number.isFinite(place.lng)) {
      const distanceMeters = calculateDistanceMeters(userLocation.value, {
        lat: place.lat,
        lng: place.lng,
      })
      return { ...place, distanceMeters }
    }
    return { ...place, distanceMeters: null }
  })
})

const selectedCategorySet = computed(() => new Set(selectedCategories.value))

const filteredPlaces = computed(() => {
  if (selectedCategories.value.length === 0) return []

  const byCategory = placesWithDistance.value.filter((place) =>
    selectedCategorySet.value.has(place.categoryKey),
  )

  const withDistance = byCategory.filter((place) => typeof place.distanceMeters === 'number')
  const withoutDistance = byCategory.filter((place) => typeof place.distanceMeters !== 'number')

  const radiusMatched = withDistance.filter((place) =>
    isPlaceWithinSelectedRadiusBand(place.distanceMeters),
  )
  if (radiusMatched.length > 0) {
    return radiusMatched.sort((a, b) => a.distanceMeters - b.distanceMeters)
  }

  // If distance is unavailable for all places, keep a stable name-sorted fallback list.
  if (!withDistance.length) return withoutDistance.sort((a, b) => a.name.localeCompare(b.name))

  return []
})

const placesWithinRadius = computed(() => {
  const byRadius = placesWithDistance.value.filter(
    (place) =>
      typeof place.distanceMeters === 'number' &&
      isPlaceWithinSelectedRadiusBand(place.distanceMeters),
  )
  if (byRadius.length > 0) return byRadius
  if (!placesWithDistance.value.some((place) => typeof place.distanceMeters === 'number')) {
    return placesWithDistance.value
  }
  return []
})

const categoryCounts = computed(() => {
  const counts = {}
  CATEGORY_OPTIONS.forEach((option) => {
    counts[option.key] = 0
  })
  placesWithinRadius.value.forEach((place) => {
    counts[place.categoryKey] = (counts[place.categoryKey] || 0) + 1
  })
  return counts
})

const totalPlaces = computed(() => filteredPlaces.value.length)
const totalPages = computed(() => Math.max(1, Math.ceil(totalPlaces.value / PLACES_PER_PAGE)))

const pagedPlaces = computed(() => {
  const start = (currentPage.value - 1) * PLACES_PER_PAGE
  return filteredPlaces.value.slice(start, start + PLACES_PER_PAGE)
})

const showSelectCategoryHint = computed(() => selectedCategories.value.length === 0)
const showNoMatchHint = computed(
  () =>
    selectedCategories.value.length > 0 &&
    !isLoadingPlaces.value &&
    totalPlaces.value === 0 &&
    !loadError.value &&
    !!userLocation.value,
)

const canExpandToExceed2Km = computed(() => showNoMatchHint.value && selectedRadius.value < 3000)
const isDetailPanelVisible = computed(() => detailPanelState.value !== 'closed')
const isDetailCategoryRich = computed(
  () =>
    !!activeDetailPlace.value &&
    ['artworks_fountains', 'memorials_sculptures'].includes(activeDetailPlace.value.categoryKey),
)
const activeMapPlaceId = ref('')
const activeMapPlace = computed(() => {
  if (!activeMapPlaceId.value) return null
  return filteredPlaces.value.find((place) => place.id === activeMapPlaceId.value) || null
})
const mapRenderablePlaces = computed(() =>
  filteredPlaces.value
    .filter((place) => Number.isFinite(place.lat) && Number.isFinite(place.lng))
    .slice(0, MAX_MAP_MARKERS),
)
const isActiveMapPlaceRich = computed(
  () =>
    !!activeMapPlace.value &&
    ['artworks_fountains', 'memorials_sculptures'].includes(activeMapPlace.value.categoryKey),
)
const isActiveMapPlaceLoading = computed(
  () => !!activeMapPlace.value && loadingDetailIds.value.has(activeMapPlace.value.id),
)
let placesRequestSeq = 0

watch(filteredPlaces, () => {
  if (currentPage.value > totalPages.value) currentPage.value = 1
  if (
    activeMapPlaceId.value &&
    !filteredPlaces.value.some((place) => place.id === activeMapPlaceId.value)
  ) {
    activeMapPlaceId.value = ''
  }
})

watch(
  [userLocation, selectedRadius],
  () => {
    if (isLoadingPlaces.value) return
    loadPlaces()
  },
  { deep: true },
)

watch(
  () => selectedCategories.value.includes('cafes_restaurants'),
  (hasCafeCategory, previousHasCafeCategory) => {
    if (!hasCafeCategory || hasCafeCategory === previousHasCafeCategory) return
    if (isLoadingPlaces.value) return
    loadPlaces()
  },
)

function toggleCategory(categoryKey) {
  if (selectedCategorySet.value.has(categoryKey)) {
    selectedCategories.value = selectedCategories.value.filter((item) => item !== categoryKey)
  } else {
    selectedCategories.value = [...selectedCategories.value, categoryKey]
  }
  currentPage.value = 1
}

function selectRadius(radiusMeters) {
  if (selectedRadius.value === radiusMeters) return
  selectedRadius.value = radiusMeters
  currentPage.value = 1
}

function goToPage(page) {
  if (page < 1 || page > totalPages.value) return
  currentPage.value = page
}

function expandTo2Km() {
  selectRadius(3000)
}

function openIdeasModal() {
  isIdeasModalOpen.value = true
  ideasStep.value = 1
  ideasTransportMode.value = ''
  ideasCategoryAnswers.value = []
}

function closeIdeasModal() {
  isIdeasModalOpen.value = false
}

function goToIdeasStep(step) {
  ideasStep.value = step
}

function toggleIdeasCategory(choiceKey) {
  if (ideasCategoryAnswers.value.includes(choiceKey)) {
    ideasCategoryAnswers.value = ideasCategoryAnswers.value.filter((item) => item !== choiceKey)
    return
  }
  ideasCategoryAnswers.value = [...ideasCategoryAnswers.value, choiceKey]
}

async function applyIdeasAnswers() {
  if (!ideasCategoryAnswers.value.length || !ideasTransportMode.value) return

  if (addressQuery.value.trim()) {
    await applyAddressFilter()
    if (addressFilterError.value) return
  }

  const resolvedCategories = IDEAS_CATEGORY_CHOICES.filter((item) =>
    ideasCategoryAnswers.value.includes(item.key),
  ).map((item) => item.categoryKey)
  const uniqueCategoryKeys = [...new Set(resolvedCategories)]
  if (!uniqueCategoryKeys.length) return

  selectedCategories.value = uniqueCategoryKeys
  selectRadius(ideasTransportMode.value === 'walking' ? 500 : 3000)
  currentPage.value = 1
  closeIdeasModal()
}

async function loadPlaceDetail(place) {
  if (!place?.id || loadingDetailIds.value.has(place.id) || detailById.value.has(place.id)) return
  loadingDetailIds.value.add(place.id)
  try {
    const configuredBase = String(import.meta.env.VITE_DISCOVER_PLACES_API_BASE_URL || '').trim()
    const baseUrl = configuredBase || DISCOVER_API_BASE_URL
    const detailEndpoint =
      place.sourceType === 'venues'
        ? `/venues/${encodeURIComponent(place.sourceId)}`
        : `/places/${encodeURIComponent(place.sourceId)}`
    const detailUrl = new URL(detailEndpoint, baseUrl).toString()
    const response = await fetch(detailUrl)
    if (!response.ok) throw new Error(`Failed to load place detail (${response.status})`)
    const detailPayload = parsePlacesPayload(await response.json())[0]
    const normalizedDetail = normalizePlace(detailPayload, 0, place.sourceType || 'places')
    if (!normalizedDetail) return

    detailById.value.set(place.id, normalizedDetail)
    allPlaces.value = allPlaces.value.map((item) =>
      item.id === place.id
        ? {
            ...item,
            ...normalizedDetail,
            categoryKey: item.categoryKey,
            categoryLabel: item.categoryLabel,
            icon: item.icon,
          }
        : item,
    )
    if (activeDetailPlace.value?.id === place.id) {
      activeDetailPlace.value =
        allPlaces.value.find((item) => item.id === place.id) || activeDetailPlace.value
    }
  } catch (error) {
    console.error('[Discover Places] Failed to load place detail:', error)
  } finally {
    loadingDetailIds.value.delete(place.id)
  }
}

let detailTransitionTimeoutId = null

function clearDetailTransitionTimeout() {
  if (detailTransitionTimeoutId !== null) {
    window.clearTimeout(detailTransitionTimeoutId)
    detailTransitionTimeoutId = null
  }
}

function openDetailPanel(place) {
  clearDetailTransitionTimeout()
  directionsError.value = ''
  activeDetailPlace.value = place
  detailPanelState.value = 'opening'
  requestAnimationFrame(() => {
    detailPanelState.value = 'open'
  })
}

function closeDetailPanel() {
  if (detailPanelState.value === 'closed' || detailPanelState.value === 'closing') return

  clearDetailTransitionTimeout()
  detailPanelState.value = 'closing'
  detailTransitionTimeoutId = window.setTimeout(() => {
    detailPanelState.value = 'closed'
    activeDetailPlace.value = null
    directionsError.value = ''
    if (pendingDetailPlace.value) {
      const next = pendingDetailPlace.value
      pendingDetailPlace.value = null
      openDetailPanel(next)
    }
  }, 320)
}

function buildDirectionsUrl(place) {
  const resolved = router.resolve({
    path: '/my-routes',
    query: {
      destination: place.name,
      destinationAddress: place.address || place.name,
      destinationLat: Number.isFinite(place.lat) ? String(place.lat) : undefined,
      destinationLng: Number.isFinite(place.lng) ? String(place.lng) : undefined,
    },
  })
  return `${window.location.origin}${resolved.href}`
}

function goToDirectionsForPlace(place) {
  router.push({
    path: '/my-routes',
    query: {
      destination: place.name,
      destinationAddress: place.address || place.name,
      destinationLat: Number.isFinite(place.lat) ? String(place.lat) : undefined,
      destinationLng: Number.isFinite(place.lng) ? String(place.lng) : undefined,
    },
  })
}

function openDirections() {
  if (!activeDetailPlace.value) return
  directionsError.value = ''
  const popup = window.open(buildDirectionsUrl(activeDetailPlace.value), '_blank')
  if (!popup) {
    directionsError.value = "We couldn't open directions right now. Please try again."
    return
  }
  popup.opener = null
  closeDetailPanel()
}

function onGlobalKeydown(event) {
  if (event.key !== 'Escape') return
  if (isIdeasModalOpen.value) {
    closeIdeasModal()
    return
  }
  closeDetailPanel()
}

let geocoder = null
let addressAutocomplete = null
let geoWatchId = null
let discoverMap = null
let mapMarkers = []
let placeMarkersById = new Map()

function buildMarkerIcon(color, isActive = false) {
  if (!window.google?.maps?.SymbolPath) return undefined
  return {
    path: window.google.maps.SymbolPath.CIRCLE,
    scale: isActive ? 12 : 8,
    fillColor: color,
    fillOpacity: 1,
    strokeColor: isActive ? '#111827' : '#ffffff',
    strokeWeight: isActive ? 3 : 2,
  }
}

function syncActiveMarkerVisual() {
  placeMarkersById.forEach((marker, placeId) => {
    const place = filteredPlaces.value.find((item) => item.id === placeId)
    const markerColor = mapMarkerColorByCategory[place?.categoryKey] || '#f97316'
    const isActive = activeMapPlaceId.value === placeId
    marker.setIcon(buildMarkerIcon(markerColor, isActive))
    marker.setZIndex(isActive ? 30 : 1)
    marker.setAnimation(isActive ? window.google?.maps?.Animation?.BOUNCE || null : null)
    if (isActive) {
      window.setTimeout(() => {
        if (activeMapPlaceId.value === placeId) marker.setAnimation(null)
      }, 700)
    }
  })
}

function closeMapPlaceCard() {
  activeMapPlaceId.value = ''
}

function panMapToAvoidPlaceCard(place, source = 'list') {
  if (!discoverMap || !Number.isFinite(place?.lat) || !Number.isFinite(place?.lng)) return
  const mapContainer = discoverMap.getDiv?.()
  if (!mapContainer) return

  const mapWidth = mapContainer.clientWidth || 0
  const mapHeight = mapContainer.clientHeight || 0
  const placeCard = mapContainer.closest('.map-panel')?.querySelector('.map-place-card')
  const cardWidth = placeCard?.clientWidth || Math.min(430, Math.max(0, mapWidth - 28))
  const cardHeight = placeCard?.clientHeight || Math.min(380, Math.max(0, mapHeight - 28))
  const horizontalPadding = 20
  const verticalPadding = 8
  const horizontalFactor = source === 'map' ? 0.34 : 0.42
  const minOffsetX = source === 'map' ? 100 : 130
  const maxOffsetX = source === 'map' ? 200 : 250
  const offsetX = Math.max(
    minOffsetX,
    Math.min(maxOffsetX, Math.round(cardWidth * horizontalFactor + horizontalPadding)),
  )
  const offsetY = Math.max(
    20,
    Math.min(90, Math.round(cardHeight * 0.14 + verticalPadding)),
  )

  discoverMap.panTo({ lat: place.lat, lng: place.lng })
  window.setTimeout(() => {
    if (!discoverMap || activeMapPlaceId.value !== place.id) return
    // Move selected point away from bottom-left info card footprint.
    discoverMap.panBy(-offsetX, offsetY)
  }, 80)
}

async function showMapPlaceCard(place, source = 'list') {
  if (!place?.id) return
  activeMapPlaceId.value = place.id
  await nextTick()
  panMapToAvoidPlaceCard(place, source)
  await loadPlaceDetail(place)
  await nextTick()
  panMapToAvoidPlaceCard(place, source)
}

async function focusMapOnPlace(place) {
  if (!place) return
  if (discoverMap) {
    const zoom = discoverMap.getZoom()
    const targetZoom = 16
    if (typeof zoom !== 'number' || zoom < targetZoom) discoverMap.setZoom(targetZoom)
  }
  await showMapPlaceCard(place)
}

function clearMapMarkers() {
  mapMarkers.forEach((marker) => marker.setMap(null))
  mapMarkers = []
  placeMarkersById = new Map()
}

function initDiscoverMap() {
  if (discoverMap || !mapContainerRef.value || !window.google?.maps?.Map) return
  discoverMap = new window.google.maps.Map(mapContainerRef.value, {
    center: { lat: -37.8136, lng: 144.9631 },
    zoom: 14,
    disableDefaultUI: false,
    mapTypeControl: false,
    streetViewControl: false,
    fullscreenControl: false,
  })
}

function updateDiscoverMapMarkers() {
  if (!discoverMap || !window.google?.maps?.Marker) return
  clearMapMarkers()

  const mapBounds = new window.google.maps.LatLngBounds()
  let hasAnyPoint = false

  if (
    userLocation.value &&
    Number.isFinite(userLocation.value.lat) &&
    Number.isFinite(userLocation.value.lng)
  ) {
    const userPos = { lat: userLocation.value.lat, lng: userLocation.value.lng }
    const userMarker = new window.google.maps.Marker({
      map: discoverMap,
      position: userPos,
      title: 'Your location',
      icon: {
        path: window.google.maps.SymbolPath.CIRCLE,
        scale: 8,
        fillColor: '#ef4444',
        fillOpacity: 1,
        strokeColor: '#ffffff',
        strokeWeight: 2,
      },
      zIndex: 10,
    })
    mapMarkers.push(userMarker)
    mapBounds.extend(userPos)
    hasAnyPoint = true
  }

  mapRenderablePlaces.value.forEach((place) => {
    const markerColor = mapMarkerColorByCategory[place.categoryKey] || '#f97316'
    const marker = new window.google.maps.Marker({
      map: discoverMap,
      position: { lat: place.lat, lng: place.lng },
      title: place.name,
      icon: buildMarkerIcon(markerColor, false),
    })
    marker.addListener('click', () => {
      showMapPlaceCard(place, 'map')
    })
    mapMarkers.push(marker)
    placeMarkersById.set(place.id, marker)
    mapBounds.extend(marker.getPosition())
    hasAnyPoint = true
  })

  if (hasAnyPoint) {
    discoverMap.fitBounds(mapBounds, 56)
    const zoom = discoverMap.getZoom()
    if (typeof zoom === 'number' && zoom > 16) discoverMap.setZoom(16)
  } else if (userLocation.value) {
    discoverMap.setCenter(userLocation.value)
    discoverMap.setZoom(14)
  }
  syncActiveMarkerVisual()
}

function clearGeoWatch() {
  if (geoWatchId !== null && navigator.geolocation?.clearWatch) {
    navigator.geolocation.clearWatch(geoWatchId)
    geoWatchId = null
  }
}

function watchDeviceLocation() {
  if (!navigator.geolocation?.watchPosition) return
  clearGeoWatch()
  geoWatchId = navigator.geolocation.watchPosition(
    ({ coords }) => {
      if (locationMode.value !== 'device') return
      userLocation.value = { lat: coords.latitude, lng: coords.longitude }
      locationUnavailable.value = false
    },
    () => {},
    { enableHighAccuracy: true, timeout: 15000, maximumAge: 5000 },
  )
}

async function requestBrowserLocation() {
  return new Promise((resolve) => {
    if (!navigator.geolocation) {
      locationUnavailable.value = true
      locationMode.value = 'none'
      resolve(null)
      return
    }

    navigator.geolocation.getCurrentPosition(
      ({ coords }) => {
        userLocation.value = { lat: coords.latitude, lng: coords.longitude }
        locationUnavailable.value = false
        locationMode.value = 'device'
        resolve(userLocation.value)
      },
      () => {
        locationUnavailable.value = true
        if (locationMode.value === 'device' || locationMode.value === 'none') {
          userLocation.value = null
          locationMode.value = 'none'
        }
        resolve(null)
      },
      { enableHighAccuracy: false, timeout: 8000, maximumAge: 60000 },
    )
  })
}

function setupAddressAutocomplete() {
  const input = addressInputRef.value
  if (!input || !window.google?.maps?.places) return

  addressAutocomplete = new window.google.maps.places.Autocomplete(input, {
    fields: ['geometry', 'formatted_address'],
    componentRestrictions: { country: 'au' },
  })
}

async function resolveAddressCoordinates() {
  const keyword = addressQuery.value.trim()
  if (!keyword) throw new Error('Please enter an address first.')

  const place = addressAutocomplete?.getPlace?.()
  if (place?.geometry?.location) {
    return {
      lat: place.geometry.location.lat(),
      lng: place.geometry.location.lng(),
      formattedAddress: place.formatted_address || keyword,
    }
  }

  if (!geocoder) throw new Error('Address lookup is unavailable right now.')

  const result = await geocoder.geocode({ address: keyword, region: 'au' })
  const first = result?.results?.[0]
  if (!first?.geometry?.location)
    throw new Error('Address not found. Please try a clearer address.')
  return {
    lat: first.geometry.location.lat(),
    lng: first.geometry.location.lng(),
    formattedAddress: first.formatted_address || keyword,
  }
}

async function applyAddressFilter() {
  addressFilterError.value = ''
  applyingAddressFilter.value = true
  try {
    await loadGoogleMapsApi()
    if (!geocoder && window.google?.maps?.Geocoder) geocoder = new window.google.maps.Geocoder()
    const target = await resolveAddressCoordinates()
    userLocation.value = { lat: target.lat, lng: target.lng }
    addressQuery.value = target.formattedAddress
    locationMode.value = 'address'
    locationUnavailable.value = false
    currentPage.value = 1
    clearGeoWatch()
  } catch (error) {
    addressFilterError.value = error?.message || 'Failed to apply address filter.'
  } finally {
    applyingAddressFilter.value = false
  }
}

async function useMyLocation() {
  addressFilterError.value = ''
  const position = await requestBrowserLocation()
  if (!position) {
    addressFilterError.value =
      'Unable to get your location. Please allow location access in browser settings.'
    return
  }
  currentPage.value = 1
  watchDeviceLocation()
}

async function loadPlaces() {
  const requestId = ++placesRequestSeq
  isLoadingPlaces.value = true
  loadError.value = ''
  try {
    const shouldLoadVenues = selectedCategories.value.includes('cafes_restaurants')
    const placesResponse = await fetch(buildPlacesApiUrl())
    if (!placesResponse.ok) throw new Error(`Failed to load places (${placesResponse.status})`)
    const placesPayload = parsePlacesPayload(await placesResponse.json())
    if (requestId !== placesRequestSeq) return
    const normalizedPlaces = placesPayload
      .map((item, index) => normalizePlace(item, index, 'places'))
      .filter(Boolean)

    let normalizedVenues = []
    if (shouldLoadVenues) {
      const { queryRadius } = buildQueryOptions()
      const venuesCacheKey = buildVenuesCacheKey(queryRadius)
      const cachedVenues = venuesCacheByQuery.get(venuesCacheKey)
      if (cachedVenues) {
        normalizedVenues = cachedVenues
      } else {
        const venuesResponse = await fetch(buildVenuesApiUrl())
        if (!venuesResponse.ok) throw new Error(`Failed to load venues (${venuesResponse.status})`)
        const venuesPayload = parsePlacesPayload(await venuesResponse.json())
        if (requestId !== placesRequestSeq) return
        normalizedVenues = venuesPayload
          .map((item, index) => normalizePlace(item, index, 'venues'))
          .filter(Boolean)
        venuesCacheByQuery.set(venuesCacheKey, normalizedVenues)
      }
    }

    allPlaces.value = [...normalizedPlaces, ...normalizedVenues]
    detailById.value = new Map()
  } catch (error) {
    if (requestId !== placesRequestSeq) return
    loadError.value = 'Places are temporarily unavailable. Please try again in a little while.'
    allPlaces.value = []
    console.error('[Discover Places] Failed to load places:', error)
  } finally {
    if (requestId !== placesRequestSeq) return
    isLoadingPlaces.value = false
  }
}

onMounted(async () => {
  readSessionState()
  await Promise.allSettled([loadPlaces(), requestBrowserLocation(), loadGoogleMapsApi()])
  if (window.google?.maps?.Geocoder) geocoder = new window.google.maps.Geocoder()
  await nextTick()
  if (window.google?.maps?.Map) {
    initDiscoverMap()
    updateDiscoverMapMarkers()
  } else {
    mapError.value = 'Map is temporarily unavailable.'
  }
  setupAddressAutocomplete()
  if (locationMode.value === 'device') watchDeviceLocation()
  window.addEventListener('keydown', onGlobalKeydown)
})

onUnmounted(() => {
  clearMapMarkers()
  clearDetailTransitionTimeout()
  clearGeoWatch()
  window.removeEventListener('keydown', onGlobalKeydown)
})

watch([filteredPlaces, userLocation], () => {
  updateDiscoverMapMarkers()
})

watch(activeMapPlaceId, () => {
  syncActiveMarkerVisual()
})
</script>

<template>
  <main class="discover-page">
    <section class="discover-shell">
      <header class="page-header">
        <button type="button" class="back-link" @click="router.push('/')">← Back</button>
        <h1>Discover places near you</h1>
      </header>

      <section class="discover-content">
        <aside class="map-panel">
          <div class="map-filters-overlay">
            <div class="map-filter-row category-row">
              <button
                v-for="category in CATEGORY_OPTIONS"
                :key="category.key"
                type="button"
                class="map-chip category-chip"
                :class="{ selected: selectedCategorySet.has(category.key) }"
                :aria-pressed="selectedCategorySet.has(category.key)"
                @click="toggleCategory(category.key)"
              >
                <span
                  class="chip-dot"
                  :style="{ backgroundColor: mapMarkerColorByCategory[category.key] || '#f97316' }"
                ></span>
                <span>{{ category.label }}</span>
                <span class="chip-count">{{ categoryCounts[category.key] || 0 }}</span>
              </button>
            </div>

            <div class="map-filter-row radius-row">
              <div class="radius-chip-group">
                <button
                  v-for="radius in RADIUS_OPTIONS"
                  :key="radius.meters"
                  type="button"
                  class="map-chip radius-chip"
                  :class="{ selected: selectedRadius === radius.meters }"
                  :aria-pressed="selectedRadius === radius.meters"
                  @click="selectRadius(radius.meters)"
                >
                  {{ radius.label }}
                </button>
              </div>

              <div class="map-filter-actions"></div>
            </div>
          </div>
          <div ref="mapContainerRef" class="map-canvas"></div>
          <article v-if="activeMapPlace" class="map-place-card">
            <button type="button" class="map-place-close" @click="closeMapPlaceCard">×</button>
            <div class="map-place-head">
              <div
                class="map-place-image"
                :style="{
                  backgroundImage: activeMapPlace.imageUrl ? `url(${activeMapPlace.imageUrl})` : '',
                }"
              >
                <span v-if="!activeMapPlace.imageUrl">{{ activeMapPlace.icon }}</span>
              </div>
              <div class="map-place-title">
                <div class="map-place-title-row">
                  <h3>{{ activeMapPlace.name }}</h3>
                  <button
                    type="button"
                    class="map-card-direction-btn"
                    @click="goToDirectionsForPlace(activeMapPlace)"
                  >
                    Direction
                  </button>
                </div>
                <p>{{ activeMapPlace.categoryLabel }}</p>
              </div>
            </div>

            <p v-if="activeMapPlace.description" class="map-place-desc">
              <span class="map-place-label">Description:</span>
              <span class="map-place-desc-text">{{ activeMapPlace.description }}</span>
            </p>
            <p class="map-place-line"><strong>Address:</strong> {{ activeMapPlace.address }}</p>
            <p
              v-if="userLocation && typeof activeMapPlace.distanceMeters === 'number'"
              class="map-place-line"
            >
              <strong>Distance:</strong> {{ formatDistance(activeMapPlace.distanceMeters) }}
            </p>
            <template v-if="isActiveMapPlaceRich">
              <p v-if="activeMapPlace.artistOrSubject" class="map-place-line">
                <strong>Artist / Subject:</strong> {{ activeMapPlace.artistOrSubject }}
              </p>
              <p v-if="activeMapPlace.year" class="map-place-line">
                <strong>Year:</strong> {{ activeMapPlace.year }}
              </p>
              <p v-if="activeMapPlace.workTitle" class="map-place-line">
                <strong>Work title:</strong> {{ activeMapPlace.workTitle }}
              </p>
              <p v-if="activeMapPlace.material" class="map-place-line">
                <strong>Material:</strong> {{ activeMapPlace.material }}
              </p>
            </template>
            <p v-if="isActiveMapPlaceLoading" class="map-place-loading">Loading details...</p>
          </article>
          <div v-if="mapError" class="map-fallback">{{ mapError }}</div>
        </aside>

        <section class="results-panel">
          <section class="location-toolbar">
            <div class="search-group">
              <input
                ref="addressInputRef"
                v-model="addressQuery"
                class="address-input"
                type="text"
                placeholder="E.g. Carlton"
                @keydown.enter.prevent="applyAddressFilter"
              />
              <button
                type="button"
                class="toolbar-btn filter-btn search-btn"
                :disabled="applyingAddressFilter"
                @click="applyAddressFilter"
              >
                {{ applyingAddressFilter ? 'Filtering...' : 'Search' }}
              </button>
            </div>
            <div class="location-actions-row">
              <button
                type="button"
                class="toolbar-btn location-btn location-inline-btn"
                @click="useMyLocation"
              >
                Use My Location
              </button>
              <button type="button" class="ideas-cta-btn inline-ideas-btn" @click="openIdeasModal">
                No ideas?
              </button>
            </div>
          </section>
          <p v-if="addressFilterError" class="address-error">{{ addressFilterError }}</p>

          <p class="result-count" v-if="!showSelectCategoryHint && !isLoadingPlaces">
            Showing {{ totalPlaces }} places
          </p>

          <div v-if="locationUnavailable" class="calm-banner">
            Location is currently off. Turn on device location for nearby-distance sorting.
          </div>

          <div v-if="showSelectCategoryHint" class="empty-state">
            Please select at least one category to see places.
          </div>

          <div v-else-if="loadError" class="empty-state">
            {{ loadError }}
          </div>

          <div v-else-if="showNoMatchHint" class="empty-state">
            <p class="no-match-text">
              No places match these filters. Try a wider distance or more categories.
            </p>
            <button
              v-if="canExpandToExceed2Km"
              type="button"
              class="action-link"
              @click="expandTo2Km"
            >
              Expand to exceed 2 km
            </button>
          </div>

          <div v-else-if="isLoadingPlaces" class="loading-state">Loading places...</div>

          <section v-else class="cards-wrap">
            <article
              v-for="place in pagedPlaces"
              :key="place.id"
              class="place-card"
              @click="focusMapOnPlace(place)"
            >
              <div class="card-left">
                <div
                  class="place-icon"
                  :style="{ backgroundImage: place.imageUrl ? `url(${place.imageUrl})` : '' }"
                >
                  <span v-if="!place.imageUrl">{{ place.icon }}</span>
                </div>
                <div class="place-main">
                  <span class="category-tag">{{
                    categoryMetaByKey[place.categoryKey].tagLabel
                  }}</span>
                  <h2>{{ place.name }}</h2>
                  <p
                    v-if="userLocation && typeof place.distanceMeters === 'number'"
                    class="distance-text"
                  >
                    {{ formatDistance(place.distanceMeters) }}
                  </p>
                </div>
              </div>
              <div class="card-actions">
                <button
                  type="button"
                  class="direction-btn"
                  @click.stop="goToDirectionsForPlace(place)"
                >
                  Direction
                </button>
              </div>
            </article>
          </section>

          <nav v-if="!showSelectCategoryHint && totalPlaces > PLACES_PER_PAGE" class="pagination">
            <button
              type="button"
              class="page-btn"
              :disabled="currentPage === 1"
              @click="goToPage(currentPage - 1)"
            >
              Prev
            </button>
            <span class="page-info">Page {{ currentPage }} of {{ totalPages }}</span>
            <button
              type="button"
              class="page-btn"
              :disabled="currentPage === totalPages"
              @click="goToPage(currentPage + 1)"
            >
              Next
            </button>
          </nav>
        </section>
      </section>
    </section>

    <div v-if="isIdeasModalOpen" class="ideas-overlay" @click="closeIdeasModal">
      <section
        class="ideas-modal"
        role="dialog"
        aria-modal="true"
        aria-label="Trip ideas survey"
        @click.stop
      >
        <header class="ideas-header">
          <h2>Quick ideas survey</h2>
          <button type="button" class="ideas-close-btn" @click="closeIdeasModal">Close</button>
        </header>

        <Transition name="ideas-step" mode="out-in">
          <div v-if="ideasStep === 1" :key="'step-1'" class="ideas-body">
            <p class="ideas-question">Do you prefer driving or walking?</p>
            <div class="ideas-option-group">
              <button
                type="button"
                class="ideas-option-btn"
                :class="{ selected: ideasTransportMode === 'walking' }"
                @click="ideasTransportMode = 'walking'"
              >
                Walking
              </button>
              <button
                type="button"
                class="ideas-option-btn"
                :class="{ selected: ideasTransportMode === 'driving' }"
                @click="ideasTransportMode = 'driving'"
              >
                Driving
              </button>
            </div>
            <div class="ideas-actions">
              <button
                type="button"
                class="ideas-next-btn"
                :disabled="!ideasTransportMode"
                @click="goToIdeasStep(2)"
              >
                Next
              </button>
            </div>
          </div>

          <div v-else :key="'step-2'" class="ideas-body">
            <p class="ideas-question">
              What kind of places would you enjoy exploring right now? (Multi-select)
            </p>
            <div class="ideas-option-group column">
              <button
                v-for="choice in IDEAS_CATEGORY_CHOICES"
                :key="choice.key"
                type="button"
                class="ideas-option-btn text-left"
                :class="{ selected: ideasCategoryAnswers.includes(choice.key) }"
                @click="toggleIdeasCategory(choice.key)"
              >
                {{ choice.label }}
              </button>
            </div>
            <div class="ideas-actions split">
              <button type="button" class="ideas-secondary-btn" @click="goToIdeasStep(1)">
                Back
              </button>
              <button
                type="button"
                class="ideas-next-btn"
                :disabled="!ideasCategoryAnswers.length"
                @click="applyIdeasAnswers"
              >
                Apply
              </button>
            </div>
          </div>
        </Transition>
      </section>
    </div>

    <div
      v-if="isDetailPanelVisible"
      class="details-overlay"
      :class="{ visible: detailPanelState === 'open' || detailPanelState === 'opening' }"
      @click="closeDetailPanel"
    >
      <aside
        class="details-panel"
        :class="{ visible: detailPanelState === 'open' || detailPanelState === 'opening' }"
        role="dialog"
        aria-modal="true"
        aria-label="Place details panel"
        @click.stop
      >
        <header class="details-panel-header">
          <div class="details-header-title">
            <h2>{{ activeDetailPlace?.name }}</h2>
            <p>{{ activeDetailPlace?.categoryLabel }}</p>
          </div>
          <button type="button" class="details-close-btn" @click="closeDetailPanel">Close</button>
        </header>

        <div class="details-panel-body">
          <section class="details-grid">
            <div v-if="activeDetailPlace?.description" class="details-item details-item-full">
              <span class="details-item-label">Description</span>
              <p class="details-item-value">{{ activeDetailPlace.description }}</p>
            </div>
            <div class="details-item">
              <span class="details-item-label">Address</span>
              <p class="details-item-value">{{ activeDetailPlace?.address }}</p>
            </div>
            <div
              v-if="activeDetailPlace && typeof activeDetailPlace.distanceMeters === 'number'"
              class="details-item"
            >
              <span class="details-item-label">Distance</span>
              <p class="details-item-value">
                {{ formatDistance(activeDetailPlace.distanceMeters) }}
              </p>
            </div>
            <template v-if="isDetailCategoryRich && activeDetailPlace">
              <div v-if="activeDetailPlace.artistOrSubject" class="details-item">
                <span class="details-item-label">Artist / Subject</span>
                <p class="details-item-value">{{ activeDetailPlace.artistOrSubject }}</p>
              </div>
              <div v-if="activeDetailPlace.year" class="details-item">
                <span class="details-item-label">Year</span>
                <p class="details-item-value">{{ activeDetailPlace.year }}</p>
              </div>
              <div v-if="activeDetailPlace.workTitle" class="details-item">
                <span class="details-item-label">Work title</span>
                <p class="details-item-value">{{ activeDetailPlace.workTitle }}</p>
              </div>
              <div v-if="activeDetailPlace.material" class="details-item">
                <span class="details-item-label">Material</span>
                <p class="details-item-value">{{ activeDetailPlace.material }}</p>
              </div>
            </template>
          </section>

          <div class="details-actions">
            <button type="button" class="details-directions-btn" @click="openDirections">
              Directions
            </button>
          </div>
          <p v-if="directionsError" class="details-error">{{ directionsError }}</p>
        </div>
      </aside>
    </div>
  </main>
</template>

<style scoped>
.discover-page {
  min-height: calc(100vh - 60px);
  background: #f4f6f5;
  padding: 18px;
  font-family:
    'Inter',
    system-ui,
    -apple-system,
    sans-serif;
  position: relative;
}

.discover-shell {
  max-width: min(98vw, 1680px);
  margin: 0 auto;
  background: #ffffff;
  border: 1px solid #dbe2de;
  border-radius: 16px;
  padding: 20px;
}

.page-header {
  display: flex;
  align-items: center;
  gap: 16px;
  border-bottom: 1px solid #e7ece8;
  padding-bottom: 14px;
  margin-bottom: 14px;
}

.back-link {
  border: none;
  background: transparent;
  color: #384046;
  font-size: 18px;
  font-weight: 700;
  cursor: pointer;
}

.page-header h1 {
  margin: 0;
  font-size: 26px;
  font-weight: 600;
  line-height: 1.25;
  color: #0f172a;
}

.location-toolbar {
  display: flex;
  flex-direction: column;
  align-items: stretch;
  gap: 8px;
  margin-bottom: 10px;
}

.location-actions-row {
  display: flex;
  align-items: center;
  gap: 8px;
}

.search-group {
  width: 100%;
  display: flex;
  align-items: stretch;
  gap: 0;
}

.address-input {
  box-sizing: border-box;
  width: 1%;
  flex: 1 1 auto;
  border: 1px solid #d1d5db;
  border-right: none;
  border-radius: 10px 0 0 10px;
  background: #ffffff;
  padding: 11px 12px;
  font-size: 14px;
  outline: none;
}

.address-input:focus {
  border-color: #059669;
  box-shadow: 0 0 0 3px #d1fae5;
}

.toolbar-btn {
  border: none;
  border-radius: 10px;
  color: #ffffff;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  padding: 0 14px;
  height: 44px;
  flex-shrink: 0;
  white-space: nowrap;
}

.toolbar-btn:disabled {
  opacity: 0.7;
  cursor: not-allowed;
}

.filter-btn {
  background: #0f766e;
}

.search-btn {
  border-radius: 0 10px 10px 0;
  min-width: 92px;
}

.location-btn {
  background: #16a34a;
}

.location-inline-btn {
  align-self: flex-start;
  border-radius: 999px;
  height: 40px;
  padding: 0 18px;
}

.address-error {
  margin: 6px 0 10px;
  color: #b91c1c;
  font-size: 12px;
}

.filters-area {
  margin-bottom: 10px;
}

.filters-content {
  display: flex;
  align-items: stretch;
  justify-content: space-between;
  gap: 14px;
}

.filters-main {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.ideas-cta-wrap {
  display: flex;
  align-items: center;
  justify-content: center;
  min-width: 140px;
}

.ideas-cta-btn {
  border: none;
  background: linear-gradient(135deg, #f97316, #ef4444);
  color: #ffffff;
  border-radius: 999px;
  min-height: 40px;
  padding: 8px 18px;
  font-size: 14px;
  font-weight: 800;
  cursor: pointer;
  box-shadow: 0 8px 20px rgba(239, 68, 68, 0.25);
  transition:
    transform 0.2s ease,
    box-shadow 0.2s ease;
}

.ideas-cta-btn:hover {
  transform: translateY(-1px) scale(1.02);
  box-shadow: 0 12px 24px rgba(239, 68, 68, 0.3);
}

.filters-row {
  display: flex;
  align-items: center;
  gap: 12px;
  flex-wrap: wrap;
}

.filters-label {
  margin: 0;
  width: 76px;
  font-size: 14px;
  font-weight: 700;
  color: #374151;
}

.chip-group {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
}

.chip {
  border: 2px solid #93bca1;
  background: #ffffff;
  color: #14532d;
  border-radius: 999px;
  min-height: 34px;
  padding: 5px 14px;
  font-size: 14px;
  font-weight: 700;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  line-height: 1.1;
}

.chip.selected {
  background: #16a34a;
  color: #ffffff;
  border-width: 2px;
  border-color: #166534;
}

.radius-chip {
  min-width: 86px;
  justify-content: center;
  text-align: center;
  padding-left: 12px;
  padding-right: 12px;
}

.result-count {
  margin: 0 0 8px;
  font-size: 14px;
  color: #475569;
  text-align: right;
}

.discover-content {
  margin-top: 12px;
  display: grid;
  grid-template-columns: minmax(0, 2.65fr) minmax(360px, 1.35fr);
  gap: 14px;
  align-items: start;
}

.map-panel {
  position: relative;
  position: sticky;
  top: 18px;
  border: 1px solid #e4eae7;
  border-radius: 14px;
  overflow: hidden;
  background: #f7faf8;
  min-height: 620px;
}

.map-canvas {
  width: 100%;
  height: 100%;
  min-height: 620px;
}

.map-filters-overlay {
  position: absolute;
  top: 12px;
  left: 12px;
  right: 12px;
  z-index: 8;
  display: flex;
  flex-direction: column;
  gap: 8px;
  pointer-events: none;
}

.map-filter-row {
  display: flex;
  gap: 8px;
  align-items: center;
  flex-wrap: wrap;
}

.map-chip,
.inline-ideas-btn {
  pointer-events: auto;
}

.map-chip {
  border: 1px solid #d1d5db;
  background: rgba(255, 255, 255, 0.95);
  color: #1f2937;
  border-radius: 999px;
  min-height: 38px;
  padding: 8px 14px;
  font-size: 14px;
  font-weight: 700;
  display: inline-flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  box-shadow: 0 4px 10px rgba(15, 23, 42, 0.12);
}

.map-chip.selected {
  border-color: #166534;
  background: #16a34a;
  color: #ffffff;
}

.map-chip.selected .chip-count {
  background: #14532d;
  color: #dcfce7;
}

.chip-dot {
  width: 10px;
  height: 10px;
  border-radius: 999px;
  border: 2px solid #ffffff;
  box-shadow: 0 0 0 1px rgba(15, 23, 42, 0.14);
}

.chip-count {
  min-width: 18px;
  height: 18px;
  border-radius: 999px;
  background: #0f766e;
  color: #ffffff;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-size: 11px;
  font-weight: 800;
  padding: 0 6px;
}

.radius-row {
  justify-content: space-between;
}

.radius-chip-group,
.map-filter-actions {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
}

.inline-ideas-btn {
  min-height: 38px;
  padding: 6px 14px;
  font-size: 13px;
}

.map-fallback {
  position: absolute;
  inset: 0;
  display: grid;
  place-items: center;
  background: rgba(248, 250, 249, 0.92);
  color: #334155;
  font-size: 14px;
  font-weight: 600;
}

.map-place-card {
  position: absolute;
  left: 14px;
  bottom: 14px;
  z-index: 9;
  width: min(430px, calc(100% - 28px));
  height: 380px;
  border-radius: 16px;
  border: 1px solid #d9e1dd;
  background: rgba(255, 255, 255, 0.98);
  box-shadow: 0 18px 34px rgba(15, 23, 42, 0.22);
  padding: 12px 14px;
  overflow-y: auto;
  overflow-x: hidden;
  overscroll-behavior: contain;
}

.map-place-close {
  position: absolute;
  top: 10px;
  right: 10px;
  border: none;
  width: 26px;
  height: 26px;
  border-radius: 999px;
  background: #eef2ef;
  color: #334155;
  font-size: 18px;
  line-height: 1;
  cursor: pointer;
}

.map-place-head {
  display: flex;
  align-items: center;
  gap: 10px;
  padding-right: 34px;
}

.map-place-image {
  width: 84px;
  height: 84px;
  border-radius: 12px;
  border: 1px solid #d1e3d5;
  background: #edf7ef;
  background-size: cover;
  background-position: center;
  display: grid;
  place-items: center;
  color: #166534;
  font-size: 30px;
  flex-shrink: 0;
}

.map-place-title-row {
  display: flex;
  align-items: center;
  gap: 8px;
  width: 100%;
}

.map-place-title h3 {
  margin: 0;
  font-size: 24px;
  line-height: 1.2;
  color: #1f2937;
  flex: 1 1 auto;
  min-width: 0;
  white-space: normal;
  overflow-wrap: anywhere;
  word-break: break-word;
}

.map-place-title p {
  margin: 4px 0 0;
  font-size: 14px;
  color: #4b5563;
  font-weight: 600;
}

.map-place-desc {
  margin: 12px 0 10px;
  font-size: 14px;
  line-height: 1.5;
  color: #1f2937;
}

.map-place-label {
  display: block;
  margin-bottom: 2px;
  font-size: 14px;
  font-weight: 700;
  color: #1f2937;
}

.map-place-desc-text {
  display: block;
  color: #6b7280;
  font-style: italic;
}

.map-place-line {
  margin: 0 0 8px;
  font-size: 14px;
  line-height: 1.45;
  color: #1f2937;
}

.map-place-line strong {
  font-size: 14px;
  font-weight: 700;
  color: #1f2937;
}

.map-place-loading {
  margin: 2px 0 0;
  color: #166534;
  font-size: 13px;
  font-weight: 600;
}

.map-card-direction-btn {
  border: none;
  border-radius: 999px;
  min-height: 34px;
  padding: 6px 10px;
  background: #166534;
  color: #ffffff;
  font-size: 12px;
  font-weight: 700;
  cursor: pointer;
  white-space: nowrap;
  flex-shrink: 0;
  margin-left: auto;
}

.map-place-actions {
  margin-top: 10px;
  display: flex;
  justify-content: flex-end;
}

.map-place-more-btn {
  border: none;
  border-radius: 999px;
  min-height: 42px;
  padding: 8px 20px;
  background: #198754;
  color: #ffffff;
  font-size: 18px;
  font-weight: 700;
  cursor: pointer;
}

.results-panel {
  min-width: 0;
  width: 100%;
}

.calm-banner {
  margin: 10px 0 12px;
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
  color: #166534;
  border-radius: 12px;
  padding: 12px 14px;
  font-size: 14px;
  line-height: 1.5;
}

.cards-wrap {
  border: 1px solid #e4eae7;
  border-radius: 14px;
  background: #f7faf8;
  padding: 10px;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.place-card {
  background: #ffffff;
  border: 1px solid #d9e1dd;
  border-radius: 14px;
  padding: 14px 16px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
  cursor: pointer;
}

.card-actions {
  display: flex;
  align-items: center;
  flex-shrink: 0;
}

.card-left {
  display: flex;
  align-items: center;
  gap: 14px;
  min-width: 0;
}

.place-icon {
  width: 74px;
  height: 74px;
  border-radius: 12px;
  background: #edf7ef;
  background-size: cover;
  background-position: center;
  border: 1px solid #d1e3d5;
  display: grid;
  place-items: center;
  color: #166534;
  font-size: 34px;
}

.place-main {
  display: flex;
  flex-direction: column;
  gap: 4px;
  min-width: 0;
}

.category-tag {
  display: inline-block;
  width: fit-content;
  padding: 3px 9px;
  border-radius: 8px;
  background: #e8f5ea;
  color: #166534;
  font-size: 12px;
  font-weight: 700;
}

.place-main h2 {
  margin: 0;
  font-size: 18px;
  line-height: 1.25;
  font-weight: 700;
  color: #111827;
}

.distance-text {
  margin: 0;
  font-size: 13px;
  color: #374151;
}

.more-info-btn {
  border: 1.5px solid #9ccaa9;
  background: #ffffff;
  color: #138c47;
  border-radius: 9px;
  min-width: 112px;
  min-height: 42px;
  padding: 7px 14px;
  font-size: 13px;
  font-weight: 700;
  cursor: pointer;
}

.direction-btn {
  border: none;
  border-radius: 999px;
  min-height: 40px;
  padding: 8px 16px;
  background: #166534;
  color: #ffffff;
  font-size: 13px;
  font-weight: 700;
  cursor: pointer;
  white-space: nowrap;
}

.empty-state,
.loading-state {
  margin: 20px 0;
  border: 1px solid #d7e2dc;
  border-radius: 12px;
  background: #f8faf9;
  color: #334155;
  padding: 18px;
  font-size: 14px;
  line-height: 1.5;
}

.empty-state p {
  margin: 0;
}

.no-match-text {
  color: #dc2626;
}

.action-link {
  margin-top: 12px;
  border: 2px solid #4b8a62;
  background: #ffffff;
  color: #14532d;
  border-radius: 8px;
  min-height: 42px;
  font-size: 14px;
  font-weight: 700;
  padding: 6px 14px;
  cursor: pointer;
}

.pagination {
  margin-top: 12px;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 10px;
}

.page-btn {
  border: 1px solid #b8c6bf;
  background: #ffffff;
  color: #334155;
  border-radius: 8px;
  min-height: 38px;
  padding: 6px 12px;
  font-size: 14px;
  font-weight: 700;
  cursor: pointer;
}

.page-btn:disabled {
  cursor: not-allowed;
  opacity: 0.55;
}

.page-info {
  font-size: 14px;
  color: #475569;
}

.details-overlay {
  position: fixed;
  inset: 0;
  background: rgba(15, 23, 42, 0.26);
  display: flex;
  justify-content: flex-end;
  opacity: 0;
  transition: opacity 0.3s ease;
  z-index: 1200;
}

.ideas-overlay {
  position: fixed;
  inset: 0;
  z-index: 1250;
  background: rgba(2, 6, 23, 0.45);
  display: grid;
  place-items: center;
  padding: 18px;
}

.ideas-modal {
  width: min(560px, 96vw);
  background: #ffffff;
  border-radius: 16px;
  border: 1px solid #f1f5f9;
  box-shadow: 0 25px 55px rgba(15, 23, 42, 0.22);
  overflow: hidden;
}

.ideas-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 16px 18px;
  border-bottom: 1px solid #e5e7eb;
}

.ideas-header h2 {
  margin: 0;
  font-size: 20px;
  color: #0f172a;
}

.ideas-close-btn {
  border: 1px solid #cbd5e1;
  background: #ffffff;
  color: #1f2937;
  border-radius: 10px;
  min-height: 36px;
  padding: 6px 12px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 700;
}

.ideas-body {
  padding: 18px;
}

.ideas-step-enter-active,
.ideas-step-leave-active {
  transition:
    opacity 0.26s ease,
    transform 0.26s ease;
}

.ideas-step-enter-from {
  opacity: 0;
  transform: translateY(10px) scale(0.99);
}

.ideas-step-leave-to {
  opacity: 0;
  transform: translateY(-8px) scale(0.99);
}

.ideas-question {
  margin: 0 0 14px;
  color: #1e293b;
  font-size: 16px;
  font-weight: 700;
}

.ideas-option-group {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.ideas-option-group.column {
  flex-direction: column;
}

.ideas-option-btn {
  border: 2px solid #cbd5e1;
  background: #ffffff;
  color: #1f2937;
  border-radius: 12px;
  min-height: 42px;
  padding: 8px 14px;
  font-size: 14px;
  font-weight: 700;
  cursor: pointer;
  transition:
    transform 0.18s ease,
    box-shadow 0.18s ease,
    border-color 0.18s ease,
    background-color 0.18s ease;
}

.ideas-option-btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 8px 18px rgba(15, 23, 42, 0.08);
}

.ideas-option-btn.text-left {
  text-align: left;
  justify-content: flex-start;
}

.ideas-option-btn.selected {
  background: #16a34a;
  border-color: #166534;
  color: #ffffff;
}

.ideas-actions {
  margin-top: 16px;
  display: flex;
  justify-content: flex-end;
}

.ideas-actions.split {
  justify-content: space-between;
}

.ideas-next-btn,
.ideas-secondary-btn {
  border: none;
  border-radius: 10px;
  min-height: 40px;
  padding: 8px 16px;
  font-size: 14px;
  font-weight: 700;
  cursor: pointer;
}

.ideas-next-btn {
  background: #166534;
  color: #ffffff;
}

.ideas-next-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.ideas-secondary-btn {
  background: #e2e8f0;
  color: #1f2937;
}

.details-overlay.visible {
  opacity: 1;
}

.details-panel {
  width: min(50vw, 760px);
  min-width: 420px;
  height: 100vh;
  background: #ffffff;
  border-left: 1px solid #dbe2de;
  transform: translateX(100%);
  transition: transform 0.32s ease;
  display: flex;
  flex-direction: column;
}

.details-panel.visible {
  transform: translateX(0);
}

.details-panel-header {
  position: sticky;
  top: 0;
  z-index: 2;
  background: #ffffff;
  border-bottom: 1px solid #e5e7eb;
  padding: 18px 20px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}

.details-header-title h2 {
  margin: 0;
  font-size: 32px;
  line-height: 1.2;
  color: #0f172a;
}

.details-header-title p {
  margin: 6px 0 0;
  font-size: 18px;
  color: #334155;
}

.details-close-btn {
  min-width: 88px;
  min-height: 44px;
  border-radius: 10px;
  border: 1px solid #cbd5e1;
  background: #ffffff;
  color: #1f2937;
  font-size: 18px;
  font-weight: 700;
  cursor: pointer;
}

.details-panel-body {
  padding: 20px;
  overflow-y: auto;
}

.details-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 16px;
}

.details-item {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  padding: 14px;
}

.details-item-full {
  grid-column: 1 / -1;
}

.details-item-label {
  display: block;
  font-size: 18px;
  font-weight: 700;
  color: #14532d;
  margin-bottom: 8px;
}

.details-item-value {
  margin: 0;
  font-size: 18px;
  line-height: 1.55;
  color: #1f2937;
}

.details-actions {
  margin-top: 20px;
  display: flex;
  gap: 12px;
  align-items: center;
}

.details-directions-btn {
  min-height: 46px;
  padding: 8px 20px;
  border: none;
  border-radius: 10px;
  background: #16a34a;
  color: #ffffff;
  font-size: 18px;
  font-weight: 700;
  cursor: pointer;
}

.details-secondary-link {
  min-height: 44px;
  padding: 8px 16px;
  border: 1px solid #cbd5e1;
  border-radius: 10px;
  background: #ffffff;
  color: #1f2937;
  font-size: 18px;
  font-weight: 700;
  cursor: pointer;
}

.details-error {
  margin: 12px 0 0;
  font-size: 18px;
  color: #b45309;
  line-height: 1.5;
}

@media (max-width: 900px) {
  .page-header h1 {
    font-size: 22px;
  }

  .place-main h2 {
    font-size: 18px;
  }

  .place-card {
    flex-direction: column;
    align-items: flex-start;
  }

  .location-toolbar {
    flex-wrap: nowrap;
  }

  .filters-content {
    flex-direction: column;
  }

  .discover-content {
    grid-template-columns: 1fr;
  }

  .map-panel {
    position: relative;
    top: 0;
    min-height: 420px;
  }

  .map-canvas {
    min-height: 420px;
  }

  .map-place-card {
    width: calc(100% - 16px);
    left: 8px;
    right: 8px;
    bottom: 8px;
    height: 330px;
    padding: 10px 12px;
  }

  .map-place-image {
    width: 72px;
    height: 72px;
  }

  .map-place-title h3 {
    font-size: 20px;
  }

  .map-filters-overlay {
    position: static;
    margin: 8px;
    background: rgba(255, 255, 255, 0.88);
    border: 1px solid #e2e8f0;
    border-radius: 12px;
    padding: 8px;
    pointer-events: auto;
  }

  .ideas-cta-wrap {
    justify-content: flex-start;
    min-width: 0;
  }

  .details-panel {
    width: min(100vw, 680px);
    min-width: 320px;
  }

  .details-grid {
    grid-template-columns: 1fr;
  }

  .details-header-title h2 {
    font-size: 24px;
  }
}
</style>
