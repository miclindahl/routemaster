<script setup>
import { ref, onMounted } from 'vue'
//import { searchBoxElement } from '../lib/mapbox'
import AddressSearchBox from '/src/components/AddressSearchBox.vue';



const API_URL = 'http://127.0.0.1:8000' // Replace with your actual API URL



const fullAddress = ref('')
const coordinates = ref('')
const selectedTimeWindow = ref(null)


const timeWindows = ref([])

async function getTimeWindows() {
  const visit_request = {
    address: fullAddress.value,
    long: coordinates.value[0],
    lat: coordinates.value[1]
  }
  try {
    const response = await fetch(`${API_URL}/get-time-windows`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify(visit_request),
      });

    if (!response.ok) {
      throw new Error('Network response was not ok')
    }
    const data = await response.json()
    console.log(data)

    const formated_time_windows = [] 
    for (let time_window in data['time_windows']) {
      const start = new Date(data['time_windows'][time_window][0])
      const end = new Date(data['time_windows'][time_window][1])
      const window = {label: `${start.getDate()}. ${start.getMonth()} ${start.getHours()}:00 - ${end.getHours()}:00`, id: time_window, from: start, to: end}
      formated_time_windows.push(window)
    }

    //const dateTimeWindows = data['time_windows'].map(window => window.map(timeStringarray => ({label:timeStringarray[0], id:timeStringarray[0], from: new Date(timeStringarray[0]) , to: new Date(timeStringarray[1])})));
    console.log(formated_time_windows)
    // TODO some formating to display nicely
    timeWindows.value = formated_time_windows
  } catch (error) {
    console.error('There was a problem fetching the time windows:', error)
  }
}

async function update_addressdata(value) {
    fullAddress.value = value.fullAddress
    coordinates.value = value.coordinates
    // enable button
}

</script>

<template>
    <div class="card flex justify-content-center">
        <Stepper linear>
            <StepperPanel header="Beskriv problemet">
                <template #content="{ nextCallback }">
                    <div class="flex flex-column h-12rem">
                        <Textarea placeholder="Beskriv dit problem her..." rows="5" class="w-full"></Textarea>
                    </div>
                    <div class="flex pt-4 justify-content-end">
                        <Button label="Næste" icon="pi pi-arrow-right" iconPos="right" @click="nextCallback" />
                    </div>
                </template>
            </StepperPanel>
            <StepperPanel header="Find tidspunkt">
                <template #content="{ prevCallback, nextCallback }">
                    <div class="flex flex-column h-12rem">
                        <AddressSearchBox @address-found="update_addressdata" class="mt-4" />
                        <Button label="Se tilgængelige tidspunkter" class="mt-2" @click="getTimeWindows" :disabled="!coordinates" />
                    </div>
                    <div class="flex flex-column mt-4">
                        <label for="timeWindowSelect" class="block text-lg font-medium text-gray-700">Vælg et tidspunkt der passer dig</label>
                          <SelectButton v-model="selectedTimeWindow" :options="timeWindows" optionLabel="label" />
                    </div>
                    <div class="flex pt-4 justify-content-between">
                        <Button label="Tilbage" severity="secondary" icon="pi pi-arrow-left" @click="prevCallback" />
                        <Button label="Næste" icon="pi pi-arrow-right" iconPos="right" @click="nextCallback" />
                    </div>
                </template>
            </StepperPanel>
            <StepperPanel header="Giv kontaktoplysninger">
                <template #content="{ prevCallback, nextCallback }">
                  <div class="flex flex-column h-12rem">
                    <InputText placeholder="Indtast dit navn..." class="mt-4" />
                    <InputText placeholder="Indtast dit telefonnummer..." class="mt-2" />
                    <InputText placeholder="Indtast din e-mail adresse..." class="mt-2" />
                  </div>
                  <div class="flex pt-4 justify-content-between">
                        <Button label="Tilbage" severity="secondary" icon="pi pi-arrow-left" @click="prevCallback" />
                        <Button label="Næste" icon="pi pi-arrow-right" iconPos="right" @click="nextCallback" />
                </div>
                </template>
            </StepperPanel>
            <StepperPanel header="Bekræftigelse">
                <template #content="{ prevCallback }">
                  <div class="flex flex-column h-12rem">
                    <p>Vi kommer snart!</p>
                  </div>
                  <div class="flex pt-4 justify-content-start">
                    <Button label="Back" severity="secondary" icon="pi pi-arrow-left" @click="prevCallback" />
                  </div>
                </template>
            </StepperPanel>
        </Stepper>
    </div>
</template>

<style scoped>
.p-stepper {
    flex-basis: 50rem;
}
</style>