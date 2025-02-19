<template>
  <q-page padding>
    <div class="row q-col-gutter-md">
      <div v-for="hotel in hotelList" :key="hotel.id" class="col-auto" style="display: inline-block; width: auto;"      >
        <q-card class="cursor-pointer hotel-card" @click="openDrawer(hotel)">
          <q-img :src="hotel.thumb" height="200px">
            <div class="absolute-top-right q-pa-sm">
              <q-btn 
                round 
                flat 
                :icon="hotel.favorite ? 'favorite' : 'favorite_border'"
                color="white"
                class="bg-grey-7"
              />
            </div>
          </q-img>

          <q-card-section>
            <div class="text-h6">{{ hotel.name }}</div>
            <div class="text-subtitle2">
              {{ hotel.address.city }}, {{ hotel.address.state }}
            </div>
            <div class="row items-center q-mt-sm">
              <q-rating
                :model-value="Number(hotel.stars)"
                size="1.5em"
                color="amber"
                readonly
              />
            </div>
            <div class="text-h6 q-mt-sm">{{ formatPrice(hotel.price) }}</div>
          </q-card-section>
        </q-card>
      </div>
    </div>

    <q-drawer
      v-model="drawerOpen"
      side="right"
      bordered
      :width="400"
      :breakpoint="600"
      overlay
    >
      <template v-if="currentHotel">
        <q-card flat style="max-width: 100%;">
          <q-card-section>
            <div class="row justify-around items-center">
              <q-rating
                :model-value="Number(currentHotel.stars)"
                size="2em"
                color="amber"
                readonly
              />
            </div>

            <div class="text-h5 q-mt-sm">{{ currentHotel.name }}</div>
            <div class="text-subtitle2">{{ currentHotel.address.fullAddress }}</div>
          </q-card-section>

          <div class="absolute-top text-center full-width q-pa-sm">
            {{ slide + 1 }} / {{ currentHotel.images.length }}
          </div>

          <q-carousel
            swipeable
            animated
            v-model="slide"
            height="300px"
            :thumbnails="true"
          >
            <q-carousel-slide 
              v-for="(image, index) in currentHotel.images"
              :key="index"
              :name="index"
              :img-src="image"
            />
          </q-carousel>

          <q-card-section>
            <div class="row q-gutter-sm">
              <q-chip
                v-for="amenity in currentHotel.amenities"
                :key="amenity.key"
                icon="check"
                color="primary"
                text-color="white"
              >
                {{ amenity.label }}
              </q-chip>
            </div>

            <p class="q-mt-md">{{ currentHotel.description }}</p>
          </q-card-section>
        </q-card>
      </template>
    </q-drawer>
  </q-page>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import hotelData from '../../database/hotel.json'
import type { Hotel } from '../types/hotel'

const drawerOpen = ref(false)
const slide = ref(0)
const currentHotel = ref<Hotel | null>(null)


const hotelList = computed(() => {
  if(hotelData[0])
    return hotelData[0].hotels as Hotel[]
  return []
})


const openDrawer = (hotel: Hotel) => {
  currentHotel.value = hotel
  drawerOpen.value = true
  slide.value = 0
}

const getAmenityIcon = (key: string): string => {
  const icons: Record<string, string> = {
    WI_FI: 'wifi',
    RESTAURANT: 'restaurant',
    ROOM_SERVICE: 'room_service'
  }
  return icons[key] || 'check'
}

const formatPrice = (price: number): string => {
  return new Intl.NumberFormat('pt-BR', {
    style: 'currency',
    currency: 'BRL'
  }).format(price)
}

</script>

<style>
.hotel-card {
  transition: transform 0.2s;
}

.hotel-card:hover {
  transform: translateY(-5px);
}
</style>