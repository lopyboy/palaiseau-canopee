<template>
  <div class="flex flex-row justify-center items-center p-4 bg-gray-900 text-black w-full h-32">
    <img src="/img/logo.png" alt="Logo" class="h-full" />
    <button
      @click="$emit('toggle-satellite')"
      class="ml-8 bg-white text-black py-2 px-4 rounded cursor-pointer hover:bg-gray-300 transition duration-300 ease-in-out"
    >
      Switch carte satellite
    </button>
  </div>
  <div class="flex flex-col jusify-center items-center w-full h-40 mt-20 mb-8">
    <p class="text-4xl text-center text-white">
      Palaiseau - Canopée
    </p>
    <div class="w-1/2 mt-4">
      <p class="text-lg text-white text-center">
        Canopée est un projet étudiant de CentraleSupélec visant à aider les mairies locales dans leurs <span class="font-bold">prises de décisions</span> concernant l'adaptation au changement climatique'.
      </p>
    </div>
    <button class="bg-white text-black py-2 px-4 rounded mt-4 cursor-pointer hover:bg-gray-300 transition duration-300 ease-in-out">
      About us
    </button>
  </div>

</template>