<script setup>
import { ref } from 'vue'
import MapContainer from '@/components/MapContainer.vue'
import Topbar from '@/components/Topbar.vue'

const isSatellite = ref(false)
</script>

<template>
  <Topbar @toggle-satellite="isSatellite = !isSatellite" />
  <div class="flex-1 mx-auto p-4 w-full">
    <MapContainer :is-satellite="isSatellite" />
  </div>
</template>