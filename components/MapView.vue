<template>
  <div class="relative w-full h-full">
    <div id="map" class="w-full h-full z-0 rounded-md"></div>
    <div v-if="loading" class="absolute inset-0 flex items-center justify-center bg-black bg-opacity-30 z-10">
      <UIcon name="i-lucide-refresh-cw" class="text-white animate-spin h-8 w-8" />
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, watch, nextTick } from 'vue';

// Define props
const props = defineProps({
  layers: {
    type: Object,
    default: () => ({})
  },
  isSatellite: {
    type: Boolean,
    default: false
  }
});

// Define reactive state
const loading = ref(true);
const map = ref(null);
const geoRasterLayers = ref({});
const L = ref(null); // Store Leaflet library reference
const GeoRasterLayer = ref(null); // Store GeoRasterLayer constructor
const initialBoundsFit = ref(false); // Track if initial bounds fit is done

const baseLayer = ref(null);

// Function to set the base layer (OSM or Satellite)
const setBaseLayer = () => {
  if (!map.value || !L.value) return;
  // Remove previous base layer if exists
  if (baseLayer.value) {
    map.value.removeLayer(baseLayer.value);
    baseLayer.value = null;
  }
  // Add the correct base layer
  if (props.isSatellite) {
    baseLayer.value = L.value.tileLayer(
      'https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}',
      {
        attribution: 'Tiles &copy; Esri &mdash; Source: Esri, i-cubed, USDA, USGS, AEX, GeoEye, Getmapping, Aerogrid, IGN, IGP, UPR-EGP, and the GIS User Community'
      }
    );
  } else {
    baseLayer.value = L.value.tileLayer(
      'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',
      {
        attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
      }
    );
  }
  baseLayer.value.addTo(map.value);
};

// Function to initialize the map
const initMap = async () => {
  try {
    // We need to dynamically import Leaflet since it requires a browser environment
    L.value = await import('leaflet');
    
    // Import georaster-layer-for-leaflet
    const GeoRasterLayerModule = await import('georaster-layer-for-leaflet');
    GeoRasterLayer.value = GeoRasterLayerModule.default;
    
    // Add leaflet CSS
    if (!document.querySelector('link[href*="leaflet.css"]')) {
      const linkEl = document.createElement('link');
      linkEl.rel = 'stylesheet';
      linkEl.href = 'https://unpkg.com/leaflet@1.9.4/dist/leaflet.css';
      document.head.appendChild(linkEl);
    }
    
    // Initialize map
    map.value = L.value.map('map').setView([48.7157, 2.2512], 13); // Palaiseau coordinates

    // Add the correct base layer
    setBaseLayer();

    loading.value = false;
  } catch (error) {
    console.error('Error initializing map:', error);
    loading.value = false;
  }
};

// Function to add a georaster layer to the map
const addGeoRasterLayer = (layerName, georaster) => {
  if (!map.value || !GeoRasterLayer.value) return;
  
  // Remove existing layer if present
  removeGeoRasterLayer(layerName);
  
  // Debug information about the raster
  console.log(`GeoRaster ${layerName} info:`, {
    noDataValue: georaster.noDataValue,
    mins: georaster.mins,
    maxs: georaster.maxs,
    ranges: georaster.ranges,
    height: georaster.height,
    width: georaster.width,
  });
  
  // Get min and max values from the georaster for proper scaling
  const min = georaster.mins ? georaster.mins[0] : 0;
  const max = georaster.maxs ? georaster.maxs[0] : 1;
  const range = max - min;
  
  // Create the GeoTIFF layer with color scale based on data range
  geoRasterLayers.value[layerName] = new GeoRasterLayer.value({
    georaster,
    opacity: 0.7,
    resolution: 256,
    pixelValuesToColorFn: values => {
      const value = values[0]; // Use the first band
      
      // Handle noData, null, undefined, or NaN values
      if (value === null || value === undefined || isNaN(value) || 
          value === georaster.noDataValue) {
        return 'rgba(0,0,0,0)'; // Transparent
      }
      
      // Normalize based on actual min/max values of this georaster
      const normalizedValue = range !== 0 ? 
        Math.max(0, Math.min(1, (value - min) / range)) : 
        0.5; // Default to middle value if range is 0
      
      // Get the color based on the layer type and normalized value
      return getColorForLayer(layerName, normalizedValue, value);
    }
  });
  
  // Add the layer to the map
  geoRasterLayers.value[layerName].addTo(map.value);
  
  // Only fit bounds when initializing the map for the first time
  // and there are no other layers yet (prevents map "unscrolling")
  if (Object.keys(geoRasterLayers.value).length === 1 && !initialBoundsFit.value) {
    map.value.fitBounds(geoRasterLayers.value[layerName].getBounds());
    initialBoundsFit.value = true; // Mark that we've done the initial fit
  }
};

// Get color for layer based on normalized value and original value
const getColorForLayer = (layerName, normalizedValue, originalValue) => {
  if (layerName === 'lst') {
    // Temperature color scale (blue to red)
    const r = Math.round(255 * normalizedValue);
    const b = Math.round(255 * (1 - normalizedValue));
    const g = Math.round(100 * (1 - Math.abs(2 * normalizedValue - 1)));
    
    return `rgba(${r}, ${g}, ${b}, 0.8)`;
  } else if (layerName === 'ndvi') {
    // Vegetation color scale (brown to green)
    // Use a special scale for NDVI which typically ranges from -1 to 1
    let ndviNormalized = normalizedValue;
    
    // If the data doesn't seem to be pre-normalized
    if (originalValue < -1 || originalValue > 1) {
      // Apply different normalization for NDVI
      ndviNormalized = Math.max(0, Math.min(1, (originalValue + 1) / 2));
    }
    
    const r = Math.round(165 - 165 * ndviNormalized);
    const g = Math.round(42 + (128 * ndviNormalized));
    const b = Math.round(42 - 42 * ndviNormalized);
    
    return `rgba(${r}, ${g}, ${b}, 0.8)`;
  } else {
    // Default color scale for other layer types
    const r = Math.round(255 * normalizedValue);
    const b = Math.round(255 * (1 - normalizedValue));
    const g = Math.round(255 * (normalizedValue < 0.5 ? normalizedValue * 2 : (1 - normalizedValue) * 2));
    
    return `rgba(${r}, ${g}, ${b}, 0.8)`;
  }
};

// Function to remove a georaster layer
const removeGeoRasterLayer = (layerName) => {
  if (map.value && geoRasterLayers.value[layerName]) {
    map.value.removeLayer(geoRasterLayers.value[layerName]);
    delete geoRasterLayers.value[layerName];
  }
};

// Method to update a GeoTIFF layer
const updateGeoRasterLayer = (layerName, georaster) => {
  if (georaster) {
    addGeoRasterLayer(layerName, georaster);
  }
};

// Method to clear all GeoTIFF layers
const clearAllLayers = () => {
  Object.keys(geoRasterLayers.value).forEach(layerName => {
    removeGeoRasterLayer(layerName);
  });
};

// Initialize map on mount
onMounted(() => {
  // Using nextTick ensures the DOM is fully loaded
  nextTick(() => {
    initMap();
  });
});

// Clean up on unmount
onUnmounted(() => {
  if (map.value) {
    map.value.remove();
    map.value = null;
  }
});

// Watch for layer changes from props
watch(() => props.layers, (newLayers) => {
  if (!map.value) return;
  
  for (const [layerName, layerData] of Object.entries(newLayers)) {
    if (layerData.active && layerData.georaster) {
      addGeoRasterLayer(layerName, layerData.georaster);
    } else if (!layerData.active) {
      removeGeoRasterLayer(layerName);
    }
  }
}, { deep: true });

// Watch for base layer switch
watch(() => props.isSatellite, () => {
  setBaseLayer();
});

// Expose methods to parent components
defineExpose({
  updateGeoRasterLayer,
  removeGeoRasterLayer,
  clearAllLayers
});
</script>