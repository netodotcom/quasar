<template>
  <q-page class="bg-grey-1">
    <div class="content-wrapper">
      <!-- Header -->
      <div class="row items-center justify-between q-pa-md bg-white">
        <div class="text-h6">Reservar hotel</div>
      </div>

      <!-- Search filters -->
      <div class="row q-pa-md bg-white q-mt-sm">
        <div class="col-12 col-md-4 q-pr-md">
          <q-input v-model="searchText" outlined dense label="Destino" @update:model-value="filterPlaces">
            <template v-slot:append>
              <q-icon name="location_on" color="grey-6" />
            </template>
          </q-input>
          <q-menu v-model="showAutocomplete" fit anchor="bottom left" self="top left" v-if="filteredPlaces.length">
            <q-list style="min-width: 100%">
              <q-item v-for="place in filteredPlaces" :key="place.placeId" clickable v-close-popup
                @click="selectPlace(place)">
                <q-item-section>
                  <q-item-label>{{ place.name }}, {{ place.state.shortname }}</q-item-label>
                </q-item-section>
              </q-item>
            </q-list>
          </q-menu>
        </div>
        <div class="col-12 col-md-8">
          <div class="row items-center justify-between">
            <div class="col-auto">
              <q-btn color="primary" label="Alterar busca" no-caps unelevated @click="filterHotels" />
            </div>
            <div class="col-auto">
              <q-select v-model="sortBy" :options="[
                { label: 'Recomendados', value: 'recommended' },
                { label: 'Preço (menor para maior)', value: 'price_asc' },
                { label: 'Preço (maior para menor)', value: 'price_desc' }
              ]" label="Organizar por" outlined dense style="width: 250px" @update:model-value="sortHotels" />
            </div>
          </div>
        </div>
      </div>

      <!-- Breadcrumbs -->
      <div class="row q-px-md q-py-sm q-mt-sm">
        <q-breadcrumbs separator=">" class="text-grey-7">
          <q-breadcrumbs-el label="Início" class="cursor-pointer" />
          <q-breadcrumbs-el label="Hotéis" class="cursor-pointer" />
          <q-breadcrumbs-el :label="selectedPlaceName" />
        </q-breadcrumbs>
      </div>

      <!-- Hotel listings -->
      <div class="q-mt-sm">
        <div v-for="hotel in displayedHotels" :key="hotel.id" class="bg-white q-mb-sm">
          <div class="row q-pa-md items-center">
            <div class="col-12 col-md-3">
              <q-carousel v-model="hotel.slide" infinite :autoplay="3000" arrows navigation class="rounded-borders"
                style="height: 160px">
                <q-carousel-slide v-for="(image, index) in hotel.images" :key="index" :name="index" :img-src="image" />
              </q-carousel>
            </div>

            <div class="col-12 col-md-6 q-px-md">
              <div class="text-h6">{{ hotel.name }}</div>
              <div class="row items-center q-gutter-x-sm">
                <div class="text-body2 text-grey-7">
                  {{ hotel.address.city }}, {{ hotel.address.state }} • {{ hotel.distance || '2,57' }} km do centro
                </div>
              </div>
              <div class="row items-center q-gutter-x-sm q-mt-sm">
                <q-rating v-model="hotel.stars" :max="5" size="1em" color="amber" readonly />
                <q-separator vertical color="grey-4" />
                <div class="row items-center q-gutter-x-sm">
                  <q-icon name="wifi" size="1.2em" color="grey-7" />
                  <q-icon name="restaurant" size="1.2em" color="grey-7" />
                  <q-icon name="pool" size="1.2em" color="grey-7" />
                </div>
              </div>
              <div class="q-mt-sm">
                <q-chip v-if="hotel.hasRefundableRoom" dense outline color="positive" class="text-caption"
                  label="Reembolsável" />
              </div>
            </div>

            <div class="col-12 col-md-3 text-right">
              <div class="text-caption text-grey-7">A partir de</div>
              <div class="text-h5 text-weight-bold q-mt-xs">
                R$ {{ hotel.price?.toFixed(2) || '1.332,00' }}
              </div>
              <div class="text-caption text-grey-7 q-mb-sm">
                + R$ 444,00/noite
                <br />
                Impostos inclusos
              </div>
              <q-btn label="Selecionar" color="primary" no-caps unelevated class="q-px-xl" @click="openDrawer(hotel)" />
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Hotel details drawer -->
    <q-drawer v-model="drawerOpen" side="right" bordered :width="600" class="bg-white">
      <q-scroll-area style="height: 100%;">
        <div v-if="selectedHotel" class="q-pa-md">
          <div class="text-h5 q-mb-md">{{ selectedHotel.name }}</div>
          <q-carousel v-model="slide" swipeable animated navigation infinite :autoplay="3000" arrows
            class="rounded-borders">
            <q-carousel-slide v-for="(image, index) in selectedHotel.images" :key="index" :name="index"
              :img-src="image" />
          </q-carousel>

          <div class="q-mt-lg">
            <div class="text-body1">{{ selectedHotel.description }}</div>
            <div class="text-h6 q-mt-lg">Comodidades do Hotel</div>
            <div class="row q-mt-md q-gutter-sm">
              <q-chip v-for="(amenity, index) in selectedHotel.amenities" :key="index" outline color="primary">
                {{ amenity.label }}
              </q-chip>
            </div>
          </div>
        </div>
      </q-scroll-area>
    </q-drawer>
  </q-page>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { useQuasar } from 'quasar'

const places = ref([])
const hotels = ref([])
const searchText = ref('')
const filteredPlaces = ref([])
const showAutocomplete = ref(false)
const selectedPlace = ref(null)
const sortBy = ref('recommended')
const drawerOpen = ref(false)
const selectedHotel = ref(null)
const slide = ref(0)

const quasar = useQuasar()
const batchSize = 10
const displayedHotels = ref([])
const loading = ref(false)
const allHotelsLoaded = ref(false)

const selectedPlaceName = computed(() => {
  if (!selectedPlace.value) return ''
  return `${selectedPlace.value.name}, ${selectedPlace.value.state.shortname}`
})

const filterPlaces = () => {
  if (!searchText.value) {
    filteredPlaces.value = []
    showAutocomplete.value = false
    return
  }

  const search = searchText.value.toLowerCase()
  filteredPlaces.value = places.value.filter(place =>
    place.name.toLowerCase().includes(search) ||
    place.state.shortname.toLowerCase().includes(search)
  )
  showAutocomplete.value = true
}

const selectPlace = (place) => {
  selectedPlace.value = place
  searchText.value = `${place.name}, ${place.state.shortname}`
  showAutocomplete.value = false
}

const filteredHotels = computed(() => {
  let filtered = hotels.value

  if (selectedPlace.value) {
    filtered = filtered.filter(hotel =>
      hotel.address.city === selectedPlace.value.name
    )
  }

  return filtered
})

const filterHotels = () => {
  resetAndLoadHotels()
}

const sortHotels = () => {
  switch (sortBy.value) {
    case 'price_asc':
      hotels.value.sort((a, b) => (a.price || 0) - (b.price || 0))
      break
    case 'price_desc':
      hotels.value.sort((a, b) => (b.price || 0) - (a.price || 0))
      break
    default:
      hotels.value.sort((a, b) => a.id - b.id)
  }
  resetAndLoadHotels()
}

const openDrawer = (hotel) => {
  selectedHotel.value = hotel
  drawerOpen.value = true
}

const loadMoreHotels = () => {
  if (loading.value || allHotelsLoaded.value) return

  loading.value = true
  setTimeout(() => {
    const nextBatch = filteredHotels.value.slice(
      displayedHotels.value.length,
      displayedHotels.value.length + batchSize
    )
    displayedHotels.value = displayedHotels.value.concat(nextBatch)
    loading.value = false
    if (displayedHotels.value.length === filteredHotels.value.length) {
      allHotelsLoaded.value = true
    }
  }, 500)
}

const resetAndLoadHotels = () => {
  displayedHotels.value = []
  allHotelsLoaded.value = false
  loadMoreHotels()
}

onMounted(async () => {
  try {
    const placesResponse = await fetch('https://hebbkx1anhila5yf.public.blob.vercel-storage.com/place-0qKNl9XwCL5wEahZHNpE26GFKFH9Ae.json')
    places.value = await placesResponse.json()

    const hotelsResponse = await fetch('https://hebbkx1anhila5yf.public.blob.vercel-storage.com/hotel-HnFHndalHLGJHLP4nmbzMVfJyXbFiC.json')
    const hotelsData = await hotelsResponse.json()
    hotels.value = hotelsData.flatMap(item => item.hotels).filter(hotel => hotel)
    resetAndLoadHotels()
  } catch (error) {
    console.error('Error fetching data:', error)
  }

  quasar.scroll.setScrollPosition(document.documentElement, 0)
  window.addEventListener('scroll', handleScroll)
})

onUnmounted(() => {
  window.removeEventListener('scroll', handleScroll)
})

const handleScroll = () => {
  const scrollPosition = window.scrollY || document.documentElement.scrollTop
  const windowHeight = window.innerHeight || document.documentElement.clientHeight
  const documentHeight = document.documentElement.scrollHeight

  if (scrollPosition + windowHeight >= documentHeight - 50) {
    loadMoreHotels()
  }
}
</script>

<style scoped>
.content-wrapper {
  max-width: 1200px;
  margin: 0 auto;
  padding: 16px;
}

.q-page {
  background: #f5f5f5;
  min-height: 100vh;
}

.rounded-borders {
  border-radius: 8px;
}

.q-chip {
  font-size: 12px;
}

:deep(.q-breadcrumbs) {
  color: #666;
  font-size: 14px;
}

:deep(.q-btn) {
  border-radius: 4px;
}

:deep(.q-field__control) {
  height: 40px;
}

:deep(.q-field__label) {
  top: 10px;
}

:deep(.q-field__native) {
  padding-top: 10px;
}

:deep(.q-carousel__navigation) {
  bottom: 6px;
}

:deep(.q-carousel__navigation-inner) {
  background: rgba(0, 0, 0, 0.3);
  border-radius: 12px;
  padding: 2px 6px;
}

:deep(.q-carousel__navigation-icon--active) {
  color: white !important;
}

:deep(.q-carousel__navigation-icon) {
  font-size: 8px;
  color: rgba(255, 255, 255, 0.5);
}
</style>