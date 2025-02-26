<template>
    <q-drawer
      v-model="show"
      side="right"
      bordered
      :width="700"
      class="bg-white hotel-drawer"
    >
      <template v-if="hotel">
        <!-- Header with close button -->
        <div class="row items-center justify-between q-pa-md">
          <div class="text-h6">Detalhes do Hotel</div>
          <q-btn flat round icon="close" @click="onClose" />
        </div>
  
        <!-- Main Image Gallery -->
        <q-carousel
          v-model="currentSlide"
          swipeable
          animated
          arrows
          navigation
          infinite
          height="300px"
          class="hotel-carousel"
        >
          <q-carousel-slide
            v-for="(image, index) in hotel.images"
            :key="index"
            :name="index"
            :img-src="image"
          >
            <div class="absolute-top-right q-pa-sm">
              <q-btn
                round
                flat
                :icon="hotel.favorite ? 'favorite' : 'favorite_border'"
                color="white"
                class="bg-grey-9 bg-opacity-50"
                @click="toggleFavorite"
              />
            </div>
          </q-carousel-slide>
        </q-carousel>
  
        <div class="q-pa-md">
          <!-- Hotel Info -->
          <div class="row items-center q-mb-md">
            <div class="col">
              <h5 class="text-h5 q-my-none">{{ hotel.name }}</h5>
              <div class="text-caption text-grey-7 q-mt-sm">
                {{ hotel.address.fullAddress }}
              </div>
            </div>
            <div class="col-auto">
              <q-rating
                v-model="stars"
                :max="5"
                size="1.2em"
                color="amber"
                icon="star"
                readonly
              />
            </div>
          </div>
  
          <!-- Facilidades do hotel section -->
          <div class="q-mb-lg">
            <div class="text-subtitle1 q-mb-sm">Facilidades do hotel</div>
            <div class="row q-col-gutter-sm">
              <div
                v-for="amenity in hotel.amenities"
                :key="amenity.key"
                class="col-auto"
              >
                <q-chip
                  outline
                  class="amenity-chip"
                >
                  <q-icon
                    :name="getAmenityIcon(amenity.key)"
                    size="18px"
                    left
                  />
                  {{ amenity.label }}
                </q-chip>
              </div>
            </div>
            <q-btn
              flat
              color="primary"
              label="Mostrar todas as facilidades"
              class="q-mt-sm"
              no-caps
            />
          </div>
  
          <!-- Description -->
          <div class="q-mb-lg">
            <div class="text-subtitle1 q-mb-sm">Conheça um pouco mais</div>
            <p class="text-body2 text-grey-8">{{ hotel.description }}</p>
          </div>
  
          <!-- Additional Info -->
          <div class="row q-col-gutter-md q-mb-lg">
            <template v-if="hotel.hasBreakFast">
              <div class="col-12">
                <q-chip outline color="positive">
                  <q-icon name="restaurant" left />
                  Café da manhã incluso
                </q-chip>
              </div>
            </template>
            <template v-if="hotel.hasRefundableRoom">
              <div class="col-12">
                <q-chip outline color="info">
                  <q-icon name="assignment_return" left />
                  Cancelamento grátis disponível
                </q-chip>
              </div>
            </template>
          </div>
  
          <!-- Price and Booking -->
          <q-separator class="q-mb-lg" />
          
          <div class="row items-center justify-between">
            <div>
              <div class="text-caption text-grey-7">Diária a partir de</div>
              <!-- <div class="text-h6">R$ {{ formatPrice(hotel.price) }}</div> -->
            </div>
            <q-btn
              color="primary"
              label="Reservar agora"
              class="q-px-xl"
              no-caps
              @click="onBookNow"
            />
          </div>
        </div>
      </template>
  
      <!-- Loading State -->
      <template v-else>
        <div class="full-height column flex-center">
          <q-spinner-dots color="primary" size="40px" />
        </div>
      </template>
    </q-drawer>
  </template>
  
  <script setup lang="ts">
  import { ref, computed } from 'vue'
  import { Hotel } from '../interfaces/int'
  
  interface Props {
    modelValue: boolean
    hotel: Hotel | null
  }
  
  interface Emits {
    (e: 'update:modelValue', value: boolean): void
    (e: 'book', hotel: Hotel): void
    (e: 'favorite', hotel: Hotel): void
  }
  
  const props = defineProps<Props>()
  const emit = defineEmits<Emits>()
  
  const currentSlide = ref(0)
  const stars = computed(() => props.hotel ? parseInt(props.hotel.stars) : 0)
  
  const show = computed({
    get: () => props.modelValue,
    set: (value) => emit('update:modelValue', value)
  })
  
  const getAmenityIcon = (key: string): string => {
    const iconMap: Record<string, string> = {
      WI_FI: 'wifi',
      RESTAURANT: 'restaurant',
      ROOM_SERVICE: 'room_service',
      AC: 'ac_unit',
      POOL: 'pool',
      GYM: 'fitness_center',
      PARKING: 'local_parking'
    }
    return iconMap[key] || 'check_circle'
  }
  
  const formatPrice = (price: number): string => {
    return price?.toLocaleString('pt-BR', {
      minimumFractionDigits: 2,
      maximumFractionDigits: 2
    }) || '0,00'
  }
  
  const onClose = () => {
    show.value = false
  }
  
  const toggleFavorite = () => {
    if (props.hotel) {
      emit('favorite', props.hotel)
    }
  }
  
  const onBookNow = () => {
    if (props.hotel) {
      emit('book', props.hotel)
    }
  }
  </script>
  
  <style scoped>
  .hotel-drawer {
    height: 100vh;
    overflow-y: auto;
  }
  
  .hotel-carousel {
    height: 300px !important;
  }
  
  .amenity-chip {
    height: 32px;
  }
  
  .bg-opacity-50 {
    opacity: 0.8;
  }
  
  :deep(.q-carousel) {
    background-color: #f5f5f5;
  }
  
  :deep(.q-carousel__navigation) {
    background: rgba(0, 0, 0, 0.3);
    padding: 4px 8px;
    border-radius: 16px;
  }
  
  :deep(.q-carousel__navigation-icon) {
    font-size: 10px;
  }
  
  :deep(.q-chip) {
    font-size: 14px;
  }
  </style>