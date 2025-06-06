<template>
  <div class="flex flex-col p-4 bg-gray-900 rounded-lg">
    <div class="flex flex-col gap-4 items-center relative">
      <div class="md:w-1/3 max-w-120 absolute top-4 right-4 z-10">
        <LayerSelector @layer-toggled="handleLayerToggle" />
      </div>
      
      <div class="w-full">
        <ClientOnly>
          <div class="h-[calc(100vh-200px)]">
            <MapView 
              :layers="activeLayers" 
              :is-satellite="isSatellite"

              ref="mapViewRef"
            />
          </div>
        </ClientOnly>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive } from 'vue';
defineProps({ isSatellite: Boolean })

// Define reactive state
const mapViewRef = ref(null);
const activeLayers = reactive({});

// Handle layer toggling from LayerSelector
const handleLayerToggle = (layerData) => {
  const { name, active, georaster } = layerData;
  
  if (active && georaster) {
    // Add or update the layer
    activeLayers[name] = { active, georaster };
  } else {
    // Deactivate the layer but keep its reference
    if (activeLayers[name]) {
      activeLayers[name].active = false;
    }
  }
};
</script>
