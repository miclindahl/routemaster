<script setup>
import { ref, onMounted } from 'vue'
import { MapboxSearchBox } from '@mapbox/search-js-web'
//import { searchBoxElement } from '../lib/mapbox'
import AddressSearchBox from '/src/components/AddressSearchBox.vue';



const API_URL = 'https://api.example.com' // Replace with your actual API URL



const fullAddress = ref('')
const coordinates = ref('')
const selectedTimeWindow = ref(null)


const timeWindows = ref([
  // Example data structure
  { id: 'tw1', label: '08:00 - 09:00' },
  { id: 'tw2', label: '09:00 - 10:00' }
  // Add more time windows as needed
])
// Function to fetch time windows from an external API
async function getTimeWindows() {
  try {
    const response = await fetch(`${API_URL}/time-windows`)
    if (!response.ok) {
      throw new Error('Network response was not ok')
    }
    const data = await response.json()
    // TODO some formating to display nicely
    timeWindows.value = data // Assuming the API returns an array of time windows
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
                        <InputText placeholder="Enter your address here..." class="mt-4" />
                        <AddressSearchBox @address-found="update_addressdata" />
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