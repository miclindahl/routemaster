<script setup>
import { ref, onMounted } from 'vue'
import { supabase } from '../lib/supabaseClient'
import { MapboxSearchBox } from '@mapbox/search-js-web'
import mapboxgl from "mapbox-gl"


const searchBoxElement = new MapboxSearchBox()

searchBoxElement.accessToken = 'pk.eyJ1IjoibWljbGluZGFobCIsImEiOiJjbHkxeGM3dG0wd3Q3MmxxdTFla2ZhNm5zIn0.JcP8D0avfCwTb86c2lQFdQ'

// set the options property
searchBoxElement.options = {
  language: 'da',
  country: 'DK',
  //types: 'address',
}
mapboxgl.accessToken = 'pk.eyJ1IjoibWljbGluZGFobCIsImEiOiJjbHkxeGM3dG0wd3Q3MmxxdTFla2ZhNm5zIn0.JcP8D0avfCwTb86c2lQFdQ'



const serviceProviders = ref([])
const name = ref('')
const fullAddress = ref('')
const floor = ref('')
const phone = ref('')
const coordinates = ref('')

searchBoxElement.addEventListener('retrieve', (e) => {
  if (e.target !== e.currentTarget) return;
  const searchText = e.detail;
  const result = searchText.features[0];  
  fullAddress.value = result.properties.full_address
  coordinates.value = result.geometry.coordinates;

});


async function addServiceProvider() {
  const point_str = `POINT(${coordinates.value[0]} ${coordinates.value[1]})`
  //console.log(point_str)
  const { data, error } = await supabase.from('service_providers').insert([
    { name: name.value, address: fullAddress.value, coordinates: point_str}
  ])
  if (error) {
    console.error('error', error)
  } else {
    name.value = ''
    fullAddress.value = ''
    floor.value = ''
    phone.value = ''
    getServiceProviders()
  }
}

let map;
let markers = [];
async function getServiceProviders() {
  const { data } = await supabase.from('service_providers').select()
  serviceProviders.value = data

  markers.forEach(marker => marker.remove());
  markers = []; // Reset the markers array

  serviceProviders.value.forEach(provider => {
    const longlat = new mapboxgl.LngLat(provider.long, provider.lat)
    const marker = new mapboxgl.Marker({ "color": "#b40219" })
      .setLngLat(longlat)
      .setPopup(new mapboxgl.Popup().setHTML(`<h3>${provider.name}</h3><p>${provider.address}</p><p>${provider.phone}</p><p>${provider.floor}</p><p>${provider.created_at}</p`))
      .addTo(map);
  })

  
}

onMounted(() => {
  document.getElementById('searchbox').appendChild(searchBoxElement);
  //document.getElementById('map').appendChild(mapElement);

  map = new mapboxgl.Map({
    container: "map",
    style: "mapbox://styles/mapbox/streets-v12",
    center: [12.5697339, 55.6753132], 
    zoom: 9,
  });
  getServiceProviders()
})

</script>

<template>
<div class="grid">
    <div class="col-12">
        <div class="card p-fluid">
            <h5>Adresse</h5>
            <div id="searchbox"></div>
            {{ fullAddress }}
            <div class="field">
                    <label for="floor">Evt. etage</label>
                    <InputText v-model="floor" id="floor" type="text" />
            </div>
            <div class="field">
                    <label for="name1">Navn</label>
                    <InputText v-model="name" id="name1" type="text" />
            </div>
            <div class="field">
                    <label for="phone">Telefonnummer</label>
                    <InputText v-model="phone" id="phone" type="text" />
            </div>
            <Button @click="addServiceProvider" label="Tilføj" class="mr-2 mb-2"></Button>

            <DataTable :value="serviceProviders" tableStyle="min-width: 50rem">
                <Column field="name" header="Name"></Column>
                <Column field="address" header="Address"></Column>
                <Column field="floor" header="Floor"></Column>
            </DataTable>
        </div>
    </div>
    <div class="col-12">
        <div class="card">
            <h5>Kort</h5>
            <div class="map-container">
                <div id="map"></div>
        </div>
    </div>
    </div>
</div>
</template>

<style scoped>
@import url('https://api.mapbox.com/mapbox-gl-js/v2.4.1/mapbox-gl.css');
.map-container {
  position: relative;
  width: 100%;
  height: 95vh;
}
#map {
  position: absolute;
  top: 0;
  bottom: 0;
  width: 100%;
  border-radius: 0.25rem;
  border: 1px solid #ccc;
}
</style>