<template>
  <q-page class="bg-grey-1">
    <div class="content-wrapper">
      <div class="row items-center justify-between q-pa-md bg-white">
        <div class="text-h6">Reservar hotel</div>
      </div>

      <div class="row q-pa-md bg-white q-mt-sm">
        <div class="col-12 col-md-4 q-pr-md position-relative">
          <q-input ref="searchInput" v-model="searchText" outlined dense label="Destino"
            @update:model-value="handleInput" class="typeahead-input">
            <template v-slot:append>
              <q-icon name="location_on" color="grey-6" />
            </template>
          </q-input>
          <div v-if="suggestion" class="suggestion-text">
            {{ searchText }}{{ suggestion.substring(searchText.length) }}
          </div>
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

      <div class="row q-px-md q-py-sm q-mt-sm">
        <q-breadcrumbs separator=">" class="text-grey-7">
          <q-breadcrumbs-el label="Início" class="cursor-pointer" />
          <q-breadcrumbs-el label="Hotéis" class="cursor-pointer" />
          <q-breadcrumbs-el :label="selectedPlaceName" />
        </q-breadcrumbs>
      </div>

      <div class="q-mt-sm">
        <div v-for="hotel in displayedHotels" :key="hotel.id" class="bg-white q-mb-sm">
          <div class="row q-pa-md items-center">
            <div class="col-12 col-md-3">
              <q-carousel v-model="hotel.slide" infinite :autoplay="3000" arrows navigation class="rounded-borders"
                style="height: 200px">
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
        <div v-if="loading" class="flex flex-center q-pa-md">
          <q-spinner-dots color="primary" size="40px" />
        </div>
      </div>
    </div>

    <template>
  <HotelDetailsDrawer
    v-model="showDrawer"
    :hotel="selectedHotel"
    @book="handleBooking"
    @favorite="handleFavorite"
  />
</template>
  </q-page>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { useQuasar } from 'quasar'
import { HotelDetailsDrawer } from '../components/HotelDetails.vue'
const places = ref([])
const hotels = ref([])
const searchText = ref('')
const selectedPlace = ref(null)
const sortBy = ref('recommended')
const drawerOpen = ref(false)
const selectedHotel = ref(null)

const quasar = useQuasar()
const batchSize = 10
const displayedHotels = ref([])
const loading = ref(false)
const allHotelsLoaded = ref(false)

const selectedPlaceName = computed(() => {
  if (!selectedPlace.value) return ''
  return `${selectedPlace.value.name}, ${selectedPlace.value.state.shortname}`
})

const suggestion = ref('')


onMounted(() => {
  window.addEventListener('keydown', handleKeyDown)
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyDown)
})


const handleInput = (value) => {
  if (!value) {
    suggestion.value = ''
    selectedPlace.value = null
    return
  }

  const match = places.value.find(place =>
    place.name.toLowerCase().startsWith(value.toLowerCase())
  )

  if (match) {
    suggestion.value = match.name
    if (value.toLowerCase() === match.name.toLowerCase()) {
      selectedPlace.value = match
      filterHotels()
    }
  } else {
    suggestion.value = ''
    selectedPlace.value = null
  }
}

const handleKeyDown = (e) => {
  if (e.key === 'Tab' || e.key === 'ArrowRight') {
    if (suggestion.value) {
      searchText.value = suggestion.value
      const match = places.value.find(place =>
        place.name.toLowerCase() === suggestion.value.toLowerCase()
      )
      if (match) {
        selectedPlace.value = match
        filterHotels()
      }
      e.preventDefault()
    }
  }
}

const filteredHotels = computed(() => {
  if (!selectedPlace.value) return hotels.value

  return hotels.value.filter(hotel =>
    hotel.address.city.toLowerCase() === selectedPlace.value.name.toLowerCase()
  )
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

const hotelStars = ref(5)

const getAmenityIcon = (key) => {
  return {
    wifi: 'wifi',
    pool: 'pool',
    restaurant: 'restaurant',
    parking: 'local_parking',
    gym: 'fitness_center',
    spa: 'spa',
    breakfast: 'restaurant',
    bar: 'local_bar',
    laundry: 'local_laundry_service',
    ac: 'ac_unit',
    tv: 'tv'
  }[key] || 'check'
}

const openDrawer = (hotel) => {
  selectedHotel.value = hotel
  hotelStars.value = hotel.stars || 5
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

    if (displayedHotels.value.length >= filteredHotels.value.length) {
      allHotelsLoaded.value = true
    }
  }, 2000)
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
.position-relative {
  position: relative;
}

.suggestion-text {
  position: absolute;
  left: 12px;
  top: 65%;
  transform: translateY(-50%);
  pointer-events: none;
  color: #999;
  z-index: 1;
  white-space: nowrap;
}

.typeahead-input :deep(input) {
  position: relative;
  background: transparent;
  z-index: 2;
}

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

:deep(.q-drawer-container) {
  width: 100vw !important;
}

:deep(.q-drawer) {
  position: fixed;
  height: 100vh;
}

:deep(.q-page-container) {
  padding-right: 0 !important;
}
</style>