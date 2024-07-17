<script setup>
import { ref } from 'vue'

const timeWindows = ref([
  // Example data structure
  { id: 'tw1', label: '08:00 - 09:00' },
  { id: 'tw2', label: '09:00 - 10:00' }
  // Add more time windows as needed
])

const selectedTimeWindow = ref(null)
// Function to fetch time windows from an external API
async function getTimeWindows() {
  try {
    const response = await fetch('https://example.com/api/time_windows') // Replace with your actual API URL
    if (!response.ok) {
      throw new Error('Network response was not ok')
    }
    const data = await response.json()
    timeWindows.value = data // Assuming the API returns an array of time windows
  } catch (error) {
    console.error('There was a problem fetching the time windows:', error)
  }
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
                        <Button label="Next" icon="pi pi-arrow-right" iconPos="right" @click="nextCallback" />
                    </div>
                </template>
            </StepperPanel>
            <StepperPanel header="Find tidspunkt">
                <template #content="{ prevCallback, nextCallback }">
                    <div class="flex flex-column h-12rem">
                        <InputText placeholder="Enter your address here..." class="mt-4" />
                        <Button label="Find ledigt tidspunkt" class="mt-2" @click="getTimeWindows" />
                    </div>
                    <div class="flex flex-column mt-4">
                        <label for="timeWindowSelect" class="block text-lg font-medium text-gray-700">Vælg et tidspunkt der passer dig</label>
                          <SelectButton v-model="selectedTimeWindow" :options="timeWindows" optionLabel="label" />
                    </div>
                    <div class="flex pt-4 justify-content-between">
                        <Button label="Back" severity="secondary" icon="pi pi-arrow-left" @click="prevCallback" />
                        <Button label="Next" icon="pi pi-arrow-right" iconPos="right" @click="nextCallback" />
                    </div>
                </template>
            </StepperPanel>
            <StepperPanel header="Kontaktoplysninger">
                <template #content="{ prevCallback, nextCallback }">
                  <div class="flex flex-column h-12rem">
                    <InputText placeholder="Indtast dit navn..." class="mt-4" />
                    <InputText placeholder="Indtast dit telefonnummer..." class="mt-2" />
                    <InputText placeholder="Indtast din e-mail adresse..." class="mt-2" />
                  </div>
                  <div class="flex pt-4 justify-content-between">
                        <Button label="Back" severity="secondary" icon="pi pi-arrow-left" @click="prevCallback" />
                        <Button label="Next" icon="pi pi-arrow-right" iconPos="right" @click="nextCallback" />
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