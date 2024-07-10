<script setup>
import { ref, onMounted } from 'vue'
import { supabase } from '../lib/supabaseClient'
import mapboxgl from "mapbox-gl"



mapboxgl.accessToken = 'pk.eyJ1IjoibWljbGluZGFobCIsImEiOiJjbHkxeGM3dG0wd3Q3MmxxdTFla2ZhNm5zIn0.JcP8D0avfCwTb86c2lQFdQ'



const visits = ref([])
const schedules = ref([])
const selectedSchedule = ref(null)
const allocations = ref([])

const serviceProviders = ref([])
const selectedServiceProvider = ref(null)

function getRangeStartDate(rangeString) {
  // The range string is in the format: "[\"2024-01-01 08:00:00+00\",\"2024-01-01 18:00:00+00\")"
  // First, remove the PostgreSQL range bounds indicators and unescape the quotes
  const cleanedString = rangeString.replace(/[\[\]\(\)\"]/g, '');

  // Split the cleaned string by comma to get the start and end dates as array elements
  const dates = cleanedString.split(',');

  // Extract the start date (first element) and ensure it's trimmed
  const startDateString = dates[0].trim();

  // Convert to a JavaScript Date object
  const startDate = new Date(startDateString);
  const localDateString = startDate.toLocaleDateString(undefined, {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  });
  return localDateString;
}

async function getSchedules() {
  const { data } = await supabase.from('schedules')
  .select('id, range, assignments (id), service_provider_assignments (id, service_provider_id, service_providers (name))')
  schedules.value = data
  console.log(data)

  // add rangestartdate to each schedule
  schedules.value.forEach(schedule => {
    schedule.rangeStartDate = getRangeStartDate(schedule.range)
  });
  selectedSchedule.value = schedules.value[0]
}

async function getServiceProviders() {
  const { data } = await supabase.from('service_providers').select()
  serviceProviders.value = data
  // Filter on schedule
  selectedServiceProvider.value = serviceProviders.value[0]
}

let groupedAssignments = ref({})
async function getAssignments() {
  // Fetch allocations with related visit details
  let { data, error } = await supabase
    .from('assignments')
    .select('schedule_id, sequence, visit_id:visits(id, name, address,lat,long), service_provider_id')
    .order('sequence', { ascending: true })

  if (error) {
    console.error(error)
    return
  }

  // Group allocations by service_provider_id
  groupedAssignments.value = data.reduce((acc, assignment) => {
    if (!acc[assignment.service_provider_id]) {
      acc[assignment.service_provider_id] = []
    }
    acc[assignment.service_provider_id].push(assignment)
    return acc
  }, {})

  const assignments = groupedAssignments.value[1]
  visits.value.forEach((_, index) => {
    const lineId = `line-${index}`;
    if (map.getLayer(lineId)) {
      map.removeLayer(lineId);
      map.removeSource(lineId); // Assuming the source id is the same as the layer id
    }
  });
  //Create a new list call places that have the coordinates for each. The first element should be the service provider and then the assignments. The last should be the service provider again.
  const places = [serviceProviders.value[0], ...assignments.map(assignment => assignment.visit_id), serviceProviders.value[0]];
  console.log(places)


  assignments.forEach((assignment, index) => {
    if (index > 0) {
      const coordinates = [
        [assignments[index - 1].visit_id.long, assignments[index - 1].visit_id.lat], // Previous visit's coordinates
        [assignment.visit_id.long, assignment.visit_id.lat] // Current visit's coordinates
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
let map;
let markers = [];
async function getVisits() {
  const { data } = await supabase.from('visits').select()
  visits.value = data

  markers.forEach(marker => marker.remove());
  markers = []; // Reset the markers array


  serviceProviders.value.forEach(serviceProvider => {
    const longlat = new mapboxgl.LngLat(serviceProvider.long, serviceProvider.lat);
    const marker = new mapboxgl.Marker({ "color": "#b40219" })
      .setLngLat(longlat)
      .setPopup(new mapboxgl.Popup().setHTML(`<h3>Provider: ${serviceProvider.name}</h3><p>${serviceProvider.address}</p><p>${serviceProvider.phone}</p><p>${serviceProvider.floor}</p><p>${serviceProvider.created_at}</p>`))
      .addTo(map);
    markers.push(marker);
  });

  visits.value.forEach(visit => {
    const longlat = new mapboxgl.LngLat(visit.long, visit.lat)
    const marker = new mapboxgl.Marker()
      .setLngLat(longlat)
      .setPopup(new mapboxgl.Popup().setHTML(`<h3>${visit.name}</h3><p>${visit.address}</p><p>${visit.phone}</p><p>${visit.floor}</p><p>${visit.created_at}</p`))
      .addTo(map);
    markers.push(marker);
  })


}

async function saveSchedule() {
  // Convert visits into a list of assignments with visit_id, schedule_id, driver_id and sequence
  const assignments = visits.value.map((visit, index) => ({
    visit_id: visit.id,
    schedule_id: selectedSchedule.value.id,
    service_provider_id: selectedServiceProvider.value.id,
    sequence: index,
  }))

  // Delete all rows with existing visit_id
  const { error: deleteError } = await supabase
    .from('assignments')
    .delete()
    .in('visit_id', visits.value.map(visit => visit.id))

  const { data, error } = await supabase
    .from('assignments')
    .upsert(assignments)
  if (error) {
    console.error(error)
    return
  }
  getAssignments()

}

onMounted(() => {
  //document.getElementById('map').appendChild(mapElement);

  map = new mapboxgl.Map({
    container: "map",
    style: "mapbox://styles/mapbox/streets-v12",
    center: [12.5697339, 55.6753132],
    zoom: 9,
  });

  getSchedules()
  getServiceProviders()
  getVisits()
  getAssignments()
})



</script>

<template>
  <div class="grid">
    <div class="col-12">
      <div class="card p-fluid">
        <h5>Schedules</h5>
        <DataTable :value="schedules" tableStyle="min-width: 50rem">
          <Column field="rangeStartDate" header="Date"></Column>
          <Column header="Service Providers">
            <template #body="slotProps">
              <span v-for="(assignment, index) in slotProps.data.service_provider_assignments" :key="index">
                {{ assignment.service_providers.name }}<span v-if="index < slotProps.data.service_provider_assignments.length - 1">, </span>
              </span>
            </template>
          </Column>
          <Column field="assignments.length" header="Visits"></Column>
        </DataTable>
        <h5>Selected</h5>
        <Dropdown v-model="selectedSchedule" :options="schedules" optionLabel="rangeStartDate" placeholder="Select" />
      </div>
    </div>
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
        <h5>Service provider</h5>
        <Dropdown v-model="selectedServiceProvider" :options="serviceProviders" optionLabel="name" placeholder="Select" />
        <h5>Besøg</h5>
        <OrderList v-model="visits" listStyle="height:250px" dataKey="id" :rows="10">
          <template #header> Visits </template>
          <template #item="slotProps">
            <div>{{ slotProps.item.name }}</div>
          </template>
        </OrderList>
        <h5>Ikke allokeret</h5>
        <p>...</p>
        <Button @click="saveSchedule" label="Save" />
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