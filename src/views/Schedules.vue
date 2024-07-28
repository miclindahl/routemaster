<script setup>
import { ref, onMounted } from 'vue'
import { supabase } from '../lib/supabaseClient'
import mapboxgl from "mapbox-gl"



mapboxgl.accessToken = 'pk.eyJ1IjoibWljbGluZGFobCIsImEiOiJjbHkxeGM3dG0wd3Q3MmxxdTFla2ZhNm5zIn0.JcP8D0avfCwTb86c2lQFdQ'
const ENGINE_API_URL = 'http://127.0.0.1:8000' // Replace with your actual API URL


const schedules = ref([])
const selectedSchedule = ref(null)

let groupedServiceProviders = ref({})
const selectedServiceProvider = ref(null)

const visits = ref([])
let groupedAssignments = ref({})


let map;
let markers = [];
let lineLayers = []

function getRangeStartAndEndDate(rangeString) {
  const cleanedString = rangeString.replace(/[\[\]\(\)\"]/g, '');
  const dates = cleanedString.split(',');
  const startDateString = dates[0].trim();
  const endDateString = dates[1].trim();

  const startDate = new Date(startDateString);
  const endDate = new Date(endDateString);
  return [startDate, endDate];
}
function formatTimeRange(rangeString) {
  // if rangestring is undefined return empty string
  if (!rangeString) {
    return '';
  }

  const [startDateTime, endDateTime] = getRangeStartAndEndDate(rangeString);

  // Extract just the time part from the full date-time strings
  const startTime = new Date(startDateTime).toLocaleTimeString('da-DK', {
    hour: '2-digit',
    minute: '2-digit'
  });
  const endTime = new Date(endDateTime).toLocaleTimeString('da-DK', {
    hour: '2-digit',
    minute: '2-digit'
  });

  return `${startTime} - ${endTime}`;
}
function formatTime(timeString) {
  return new Date(timeString).toLocaleTimeString('da-DK', {
    hour: '2-digit',
    minute: '2-digit'
  });
}
function getRangeStartDate(rangeString) {
  const [startDateTime, endDateTime] = getRangeStartAndEndDate(rangeString);
  const localDateString = startDateTime.toLocaleDateString('da-DK', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  });
  return localDateString;
}

async function getSchedules() {
  const { data } = await supabase.from('schedules')
  .select('id, range, assignments (id), service_provider_assignments (id, service_provider_id, service_providers (id, name, lat, long, address))')
  schedules.value = data

  // add rangestartdate to each schedule
  schedules.value.forEach(schedule => {
    schedule.rangeStartDate = getRangeStartDate(schedule.range)
  });
  selectedSchedule.value = schedules.value[0]

  schedules.value.forEach(schedule => {
    if (!groupedServiceProviders.value[schedule.id]) {
        groupedServiceProviders.value[schedule.id] = []
    }
    schedule.service_provider_assignments.forEach(assignment => {
      groupedServiceProviders.value[schedule.id].push(assignment.service_providers)
    })
  })

  selectedServiceProvider.value = groupedServiceProviders.value[selectedSchedule.value.id][0]
}



async function getAssignments() {
  // Fetch allocations with related visit details
  let { data, error } = await supabase
    .from('assignments')
    .select('schedule_id, sequence, driving_time, waiting_time, eta, etd, visit_id:visits(id, name, address, expected_duration, time_window, lat,long), service_provider_id')
    .eq('schedule_id', selectedSchedule.value.id)
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
    acc[assignment.service_provider_id].push({...assignment.visit_id, driving_time: assignment.driving_time, waiting_time: assignment.waiting_time, eta: assignment.eta, etd: assignment.etd, sequence: assignment.sequence})
    return acc
  }, {})

  visits.value = Object.values(groupedAssignments.value).flat()
}


async function reOptimize(schedule_id) {
  try {
    const response = await fetch(`${ENGINE_API_URL}/reoptimize/${schedule_id}`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({}),
      });

    if (!response.ok) {
      throw new Error('Network response was not ok')
    }
    const data = await response.json()
    console.log(data)
  } catch (error) {
    console.error('There was a problem:', error)
  }
}

async function drawRoutes() {

  // for each layer in lineLayers, remove the layer from the map
  lineLayers.forEach(layer => {
    if (map.getLayer(layer)) {
    map.removeLayer(layer)
    map.removeSource(layer)
    }
  })

  Object.keys(groupedAssignments.value).forEach((serviceProviderId) => {
    // Get the service provider object
    const serviceProvider = groupedServiceProviders.value[selectedSchedule.value.id].find(provider => provider.id===parseInt(serviceProviderId))

    const places=[serviceProvider, ...groupedAssignments.value[serviceProviderId], serviceProvider]

    places.forEach((assignment, index) => {
      if(index>0) {
        const coordinates=[
          [places[index-1].long, places[index-1].lat],
          [assignment.long, assignment.lat] // Current visit's coordinates
        ]
        let line_id=`line-${places[index-1].id}-${assignment.id}`
        lineLayers.push(line_id)
        const lineFeature={
          type: 'Feature',
          geometry: {
            type: 'LineString',
            coordinates: coordinates,
          },
        }
        map.addLayer({
          id: line_id,
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
        })
      }
    })
  })
}

async function drawMarkers() {
  markers.forEach(marker => marker.remove());
  markers = [];
  groupedServiceProviders.value[selectedSchedule.value.id].forEach(serviceProvider => {
    const longlat = new mapboxgl.LngLat(serviceProvider.long, serviceProvider.lat);
    const marker = new mapboxgl.Marker({ "color": "#b40219" })
      .setLngLat(longlat)
      .setPopup(new mapboxgl.Popup().setHTML(`<h3>Provider: ${serviceProvider.name}</h3><p>${serviceProvider.address}</p><p>${serviceProvider.created_at}</p>`))
      .addTo(map);
    markers.push(marker);
  });

  visits.value.forEach(visit => {
    const longlat = new mapboxgl.LngLat(visit.long, visit.lat)
    const marker = new mapboxgl.Marker()
      .setLngLat(longlat)
      .setPopup(new mapboxgl.Popup().setHTML(`<h3>${visit.name}</h3><p>${visit.address}</p><p>${visit.created_at}</p`))
      .addTo(map);
    markers.push(marker);
  })
}

async function syncAssignments() {
  await getAssignments()
  drawMarkers()
  drawRoutes()
}

async function saveSchedule() {

  let visitlist = groupedAssignments.value[selectedServiceProvider.value.id]

  const assignments = visitlist.map((visit, index) => ({
    visit_id: visit.id,
    schedule_id: selectedSchedule.value.id,
    service_provider_id: selectedServiceProvider.value.id,
    sequence: index,
  }))

  // Delete all rows with existing visit_id
  const { error: deleteError } = await supabase
    .from('assignments')
    .delete()
    .in('visit_id', visitlist.map(visit => visit.id))

  const { data, error } = await supabase
    .from('assignments')
    .upsert(assignments)
  if (error) {
    console.error(error)
    return
  }
  syncAssignments()

}

onMounted(async () => {

  map = new mapboxgl.Map({
    container: "map",
    style: "mapbox://styles/mapbox/streets-v12",
    center: [12.5697339, 55.6753132],
    zoom: 9,
  });
  await getSchedules()
  syncAssignments()
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
          <Column field="assignments.length" header="Besøg"></Column>
          <Column field="assignments.length" header="Udnyttelsesgrad">
            <template #body="{ data }">
                <ProgressBar :value="(data.assignments.length / (9*data.service_provider_assignments.length)) * 100" :showValue="false" style="height: 0.5rem"></ProgressBar>
            </template>
          </Column>
        </DataTable>
        <h5>Selected</h5>
        <Dropdown v-model="selectedSchedule" :options="schedules" optionLabel="rangeStartDate" placeholder="Select" @change="syncAssignments"/>
      </div>
    </div>
    <div class="col-12">
      <template v-if="selectedSchedule">
       
      <div v-for="provider in groupedServiceProviders[selectedSchedule.id]" :key="provider.id" class="card p-fluid">
        <h5>{{provider.name}}</h5>
        Forventet arbejdstid: 9:00 - 16:53 (7:53)<br/>
        Køretid: 2:34 (xxkm) <br/>  
        <DataTable :value="groupedAssignments[provider.id]" tableStyle="min-width: 50rem">
          <Column field="sequence" header="#"></Column>
          <Column field="name" header="Name"></Column>
          <Column field="address" header="Address"></Column>
          <Column header="Tidsvindue">
            <template #body="slotProps">
              {{ formatTimeRange(slotProps.data.time_window) }}
            </template>
          </Column>
          <Column field="driving_time" header="Kørsel (m.)"></Column>
          <Column field="waiting_time" header="Ventetid (m.)">
            <template #body="slotProps">
              <span v-if="slotProps.data.waiting_time > 0">{{ slotProps.data.waiting_time }}</span>
              <span v-else></span>
            </template>
          </Column>
          <Column header="ETA">
            <template #body="slotProps">
              {{ formatTime(slotProps.data.eta) }}
            </template>
          </Column>
          <Column header="ETD">
            <template #body="slotProps">
              {{ formatTime(slotProps.data.etd) }}
            </template>
          </Column>
          <!--Column field="expected_duration" header="Duration (m.)"></Column-->
        </DataTable>
      </div>
    </template>
    </div>
    <div class="col-4">
      <div class="card">
        <h5>Service provider</h5>
        <Dropdown v-if="selectedSchedule" v-model="selectedServiceProvider" :options="groupedServiceProviders[selectedSchedule.id]" optionLabel="name" placeholder="Select" />
        <h5>Besøg</h5>
        <OrderList v-if="selectedServiceProvider" v-model="groupedAssignments[selectedServiceProvider.id]" listStyle="height:250px" dataKey="id" :rows="10">
          <template #header> Visits </template>
          <template #item="slotProps">
            <div>{{ slotProps.item.name }}</div>
          </template>
        </OrderList>
        <Button @click="saveSchedule" label="Save" />
        <Button @click="reOptimize(selectedSchedule.id)" label="Re-optimize" />
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