<template>
  <div class="p-4 bg-white rounded-lg shadow-md text-black">
    <h3 class="text-lg font-semibold mb-4">Couches de données</h3>
    
    <div v-if="loading" class="flex items-center justify-center py-4">
      <UIcon name="i-lucide-refresh-cw" class="animate-spin h-5 w-5 mr-2" />
      <span>Chargement en cours...</span>
    </div>
    
    <div v-else class="space-y-3">
      <div class="flex items-center">
        <UCheckbox 
          v-model="activeLayers.lst"
          @change="toggleLayer('lst')"
          :disabled="loadingStatus.lst"
        />
        <span class="ml-2">Land Surface Temperature (LST)</span>
        <div v-if="loadingStatus.lst" class="ml-2">
          <UIcon name="i-lucide-refresh-cw" class="animate-spin h-4 w-4" />
        </div>
      </div>
      
      <div class="flex items-center">
        <UCheckbox 
          v-model="activeLayers.ndvi"
          @change="toggleLayer('ndvi')"
          :disabled="loadingStatus.ndvi"
        />
        <span class="ml-2">Normalized Difference Vegetation Index (NDVI)</span>
        <div v-if="loadingStatus.ndvi" class="ml-2">
          <UIcon name="i-lucide-refresh-cw" class="animate-spin h-4 w-4" />
        </div>
      </div>
      
      <div class="mt-4">
        <div class="text-sm font-medium mb-1">Legends</div>
        
        <!-- LST Legend -->
        <div v-if="activeLayers.lst" class="mb-2">
          <div class="text-xs">Land Surface Temperature (LST)</div>
          <div class="flex items-center">
            <div class="w-full h-4 bg-gradient-to-r from-blue-500 via-purple-500 to-red-500 rounded"></div>
          </div>
          <div class="flex justify-between text-xs mt-1">
            <span>Low Temp</span>
            <span>High Temp</span>
          </div>
        </div>
        
        <!-- NDVI Legend -->
        <div v-if="activeLayers.ndvi" class="mb-2">
          <div class="text-xs">NDVI</div>
          <div class="flex items-center">
            <div class="w-full h-4 bg-gradient-to-r from-amber-700 via-yellow-400 to-green-600 rounded"></div>
          </div>
          <div class="flex justify-between text-xs mt-1">
            <span>Low Vegetation</span>
            <span>High Vegetation</span>
          </div>
        </div>
        
        <!-- Default legend when no layers are active -->
        <div v-if="!activeLayers.lst && !activeLayers.ndvi" class="text-xs text-gray-500">
          Select a layer to see its legend
        </div>
      </div>
    </div>
    
    <p v-if="error" class="mt-2 text-sm text-red-500">{{ error }}</p>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted } from 'vue';

// Define emits
const emit = defineEmits(['layer-toggled']);

// Define reactive state
const loading = ref(true);
const error = ref('');
const loadingStatus = reactive({
  lst: false,
  ndvi: false
});
const activeLayers = reactive({
  lst: false,
  ndvi: false
});

// Toggle a layer when checkbox is checked/unchecked
const toggleLayer = async (layerName) => {
  if (activeLayers[layerName]) {
    // Load this layer
    loadingStatus[layerName] = true;
    try {
      // Import required libraries
      const { default: parseGeoraster } = await import('georaster');
      
      // Fetch the GeoTIFF file
      const response = await fetch(`/geotiff/${layerName.toUpperCase()}.tif`);
      if (!response.ok) {
        throw new Error(`Failed to fetch ${layerName.toUpperCase()}.tif`);
      }
      
      const arrayBuffer = await response.arrayBuffer();
      const georaster = await parseGeoraster(arrayBuffer);
      
      // Emit the event with the loaded georaster
      emit('layer-toggled', {
        name: layerName,
        active: true,
        georaster
      });
    } catch (err) {
      console.error(`Error loading ${layerName} layer:`, err);
      error.value = `Failed to load ${layerName.toUpperCase()} layer`;
      activeLayers[layerName] = false;
    } finally {
      loadingStatus[layerName] = false;
    }
  } else {
    // Deactivate this layer
    emit('layer-toggled', {
      name: layerName,
      active: false
    });
  }
};

// Initialize on component mount
onMounted(() => {
  // Check if the GeoTIFF files exist in the public directory
  Promise.all([
    fetch('/geotiff/LST.tif').then(res => res.ok),
    fetch('/geotiff/NDVI.tif').then(res => res.ok)
  ])
  .then(([lstExists, ndviExists]) => {
    if (!lstExists || !ndviExists) {
      error.value = 'Some GeoTIFF files are missing from public/geotiff/ directory';
    }
  })
  .catch(err => {
    error.value = 'Error checking GeoTIFF files';
    console.error(err);
  })
  .finally(() => {
    loading.value = false;
  });
});
</script>
