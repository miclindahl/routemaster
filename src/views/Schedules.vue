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



const visits = ref([])


const dropdownValues = ref([
    { name: 'New York', code: 'NY' },
    { name: 'Rome', code: 'RM' },
    { name: 'London', code: 'LDN' },
    { name: 'Istanbul', code: 'IST' },
    { name: 'Paris', code: 'PRS' }
]);
const dropdownValue = ref(null);





let map;
let markers = [];
async function getVisits() {
  const { data } = await supabase.from('visits').select()
  visits.value = data

  markers.forEach(marker => marker.remove());
  markers = []; // Reset the markers array
  visits.value.forEach((_, index) => {
    const lineId = `line-${index}`;
    if (map.getLayer(lineId)) {
      map.removeLayer(lineId);
      map.removeSource(lineId); // Assuming the source id is the same as the layer id
    }
  });
  visits.value.forEach(visit => {
    const longlat = new mapboxgl.LngLat(visit.long, visit.lat)
    const marker = new mapboxgl.Marker()
      .setLngLat(longlat)
      .setPopup(new mapboxgl.Popup().setHTML(`<h3>${visit.name}</h3><p>${visit.address}</p><p>${visit.phone}</p><p>${visit.floor}</p><p>${visit.created_at}</p`))
      .addTo(map);
  })
  // Add a line between each visit
  visits.value.forEach((visit, index) => {
  if (index > 0) {
    const coordinates = [
      [visits.value[index - 1].long, visits.value[index - 1].lat], // Previous visit's coordinates
      [visit.long, visit.lat] // Current visit's coordinates
    ];
    const lineFeature = {
      type: 'Feature',
      geometry: {
        type: 'LineString',
        coordinates: coordinates,
      },
    };
    map.addLayer({
      id: `line-${index}`,
      type: 'line',
      source: {
        type: 'geojson',
        data: lineFeature,
      },
      layout: {
        'line-join': 'round',
        'line-cap': 'round',
      },
      paint: {
        'line-color': '#888',
        'line-width': 8,
      },
    });
  }
});
  
}

onMounted(() => {
  //document.getElementById('map').appendChild(mapElement);

  map = new mapboxgl.Map({
    container: "map",
    style: "mapbox://styles/mapbox/streets-v12",
    center: [12.5697339, 55.6753132], 
    zoom: 9,
  });
  getVisits()
})



</script>

<template>
<div class="grid">
    <div class="col-12">
        <div class="card p-fluid">
            <h5>Adresser</h5>
            <DataTable :value="visits" tableStyle="min-width: 50rem">
                <Column field="name" header="Name"></Column>
                <Column field="address" header="Address"></Column>
                <Column field="floor" header="Floor"></Column>
            </DataTable>
        </div>
    </div>
    <div class="col-4">
            <div class="card">
              <h5>Dropdown</h5>
                <Dropdown v-model="dropdownValue" :options="dropdownValues" optionLabel="name" placeholder="Select" />
                <h5>Besøg</h5>
                <OrderList v-model="visits" listStyle="height:250px" dataKey="code" :rows="10">
                    <template #header> Visits </template>
                    <template #item="slotProps">
                        <div>{{ slotProps.item.name }}</div>
                    </template>
                </OrderList>
            </div>
        </div>
    <div class="col-8">
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