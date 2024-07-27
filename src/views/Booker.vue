<script setup>
import { ref, onMounted } from 'vue'
//import { searchBoxElement } from '../lib/mapbox'
import AddressSearchBox from '/src/components/AddressSearchBox.vue';



const API_URL = 'http://127.0.0.1:8000' // Replace with your actual API URL



const fullAddress = ref('')

const coordinates = ref('')
const selectedTimeWindow = ref(null)
const name = ref('')
const phone = ref('')
const email = ref('')

const timeWindows = ref(null)
const isWaiting = ref(false)
const bookingSuccess = ref(null)

async function getTimeWindows() {
  isWaiting.value = true
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
      const date_string = start.toLocaleDateString('da-DK', { weekday: "long", day: "numeric", month: 'long' })
      const window = {label: `${date_string} ${start.getHours()}:00 - ${end.getHours()}:00`, id: time_window, from: start, to: end}
      formated_time_windows.push(window)
    }

    //const dateTimeWindows = data['time_windows'].map(window => window.map(timeStringarray => ({label:timeStringarray[0], id:timeStringarray[0], from: new Date(timeStringarray[0]) , to: new Date(timeStringarray[1])})));
    console.log(formated_time_windows)
    // TODO some formating to display nicely
    timeWindows.value = formated_time_windows
    isWaiting.value = false
  } catch (error) {
    console.error('There was a problem fetching the time windows:', error)
    isWaiting.value = false
  }
}

async function book_visit(nextCallback) {
  isWaiting.value = true
  const visit_request = {
    name: name.value,
    phone: phone.value,
    email: email.value,
    address: fullAddress.value,
    long: coordinates.value[0],
    lat: coordinates.value[1],
    window_from: selectedTimeWindow.value.from,
    window_to: selectedTimeWindow.value.to,
  }
  try {
    const response = await fetch(`${API_URL}/book-visit`, {
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
    bookingSuccess.value = true
    isWaiting.value = false
    nextCallback()
  } catch (error) {
    console.error('There was a problem booking the visit:', error)
    bookingSuccess.value = false
    isWaiting.value = false
  }

}


async function update_addressdata(value) {
    fullAddress.value = value.fullAddress
    coordinates.value = value.coordinates
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
                      <p><i v-if="isWaiting" class="pi pi-spin pi-spinner" style="font-size: 2rem"></i></p>
                      <div v-if="timeWindows">
                      <label for="timeWindowSelect" class="block text-lg font-medium text-gray-700">Vælg et tidspunkt der passer dig</label>
                          <SelectButton v-model="selectedTimeWindow" :options="timeWindows" optionLabel="label" />
                          <p>Du kan nemt rebooke tidspunktet senere, hvis tidspunktet ikke passer alligevel.</p>
                        </div> 
                      </div>
                   
                    <div class="flex pt-4 justify-content-between">
                        <Button label="Tilbage" severity="secondary" icon="pi pi-arrow-left" @click="prevCallback" />
                        <Button label="Næste" icon="pi pi-arrow-right" iconPos="right" @click="nextCallback"  :disabled="!selectedTimeWindow" />
                    </div>
                </template>
            </StepperPanel>
            <StepperPanel header="Giv kontaktoplysninger">
                <template #content="{ prevCallback, nextCallback }">
                  <div class="flex flex-column h-12rem">
                    <InputText v-model="name" placeholder="Indtast dit navn..." class="mt-4" />
                    <InputText v-model="phone" placeholder="Indtast dit telefonnummer..." class="mt-2" />
                    <InputText v-model="email" placeholder="Indtast din e-mail adresse..." class="mt-2" />
                    <p><i v-if="isWaiting" class="pi pi-spin pi-spinner" style="font-size: 2rem"></i></p>
                  </div>
                  <div class="flex pt-4 justify-content-between">
                        <Button label="Tilbage" severity="secondary" icon="pi pi-arrow-left" @click="prevCallback" />
                        <Button label="Book tiden" icon="pi pi-check" iconPos="right" @click="book_visit(nextCallback)" />
                </div>
                </template>
            </StepperPanel>
            <StepperPanel header="Bekræftigelse">
                <template #content="">
                  <div class="flex flex-column h-12rem">
                    <p style="color: green;">
                      <i class="pi pi-check" style="font-size: 24px;"></i>
                      Din tid er bekræftiget!
                    </p>
                    <p>Vi har sendt en bekræftigelse på mail og sms. Hvis du vil lave ændre så kan du nemt gøre det via det vedlagte link.</p>
                  </div>
                  <div class="flex pt-4 justify-content-start">
                    <!--Button label="Back" severity="secondary" icon="pi pi-arrow-left" @click="prevCallback" /-->
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