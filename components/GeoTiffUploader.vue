<template>
  <div class="p-4 bg-white rounded-lg shadow-md">
    <h3 class="text-lg font-semibold mb-4">Upload GeoTIFF</h3>
    <UInput
      type="file"
      accept=".tif,.tiff,image/tiff"
      @change="handleFileUpload"
      :disabled="loading"
    />
    <p v-if="error" class="mt-2 text-sm text-red-500">{{ error }}</p>
    <div class="mt-4 flex space-x-4">
      <UButton
        color="primary"
        :loading="loading"
        :disabled="!selectedFile || loading"
        @click="processFile"
      >
        Load GeoTIFF
      </UButton>
      <UButton
        v-if="isFileLoaded"
        color="secondary"
        :disabled="loading"
        @click="$emit('clear')"
      >
        Clear
      </UButton>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue';

// Define props and emits
const emit = defineEmits(['file-loaded', 'clear']);

// Define reactive state
const selectedFile = ref(null);
const loading = ref(false);
const error = ref('');
const isFileLoaded = ref(false);

// Handle file selection
const handleFileUpload = (event) => {
  const file = event.target.files[0];
  if (file) {
    selectedFile.value = file;
    error.value = '';
  }
};

// Process the selected file
const processFile = async () => {
  if (!selectedFile.value) {
    error.value = 'Please select a file';
    return;
  }

  try {
    loading.value = true;
    error.value = '';

    // Read the file as an ArrayBuffer
    const reader = new FileReader();
    
    reader.onload = async (e) => {
      try {
        // Import geotiff parser (only imported when needed)
        const { default: parseGeoraster } = await import('georaster');
        
        // Parse the GeoTIFF
        const arrayBuffer = e.target.result;
        const georaster = await parseGeoraster(arrayBuffer);

        // Emit the parsed georaster
        emit('file-loaded', {
          georaster,
          filename: selectedFile.value.name
        });
        
        isFileLoaded.value = true;
        loading.value = false;
      } catch (err) {
        console.error(err);
        error.value = 'Error processing GeoTIFF file. Check console for details.';
        loading.value = false;
      }
    };

    reader.onerror = () => {
      error.value = 'Error reading file';
      loading.value = false;
    };

    reader.readAsArrayBuffer(selectedFile.value);
  } catch (err) {
    console.error(err);
    error.value = 'An unexpected error occurred';
    loading.value = false;
  }
};
</script>
