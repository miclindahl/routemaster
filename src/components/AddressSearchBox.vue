<script setup>
const emit = defineEmits(['address-found'])


import { ref, onMounted } from 'vue'
import { MapboxSearchBox } from '@mapbox/search-js-web'
const searchBoxElement = new MapboxSearchBox()

searchBoxElement.accessToken = 'pk.eyJ1IjoibWljbGluZGFobCIsImEiOiJjbHkxeGM3dG0wd3Q3MmxxdTFla2ZhNm5zIn0.JcP8D0avfCwTb86c2lQFdQ'
searchBoxElement.options = {
  language: 'da',
  country: 'DK',
  types: 'street,address',
}

onMounted(() => {
    document.getElementById('searchbox').appendChild(searchBoxElement);
})
const fullAddress = ref('')
const coordinates = ref('')

searchBoxElement.addEventListener('retrieve', (e) => {
  if (e.target !== e.currentTarget) return;
  const searchText = e.detail;
  const result = searchText.features[0];  
  fullAddress.value = result.properties.full_address
  coordinates.value = result.geometry.coordinates;
  emit('address-found', { fullAddress: fullAddress.value, coordinates: coordinates.value })
  //console.log(fullAddress);
  //console.log(coordinates);
    // Should emit event to parent.
});
</script>


<template>  
<div id="searchbox"></div>
</template>