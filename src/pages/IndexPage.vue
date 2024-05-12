// some Vue file
// remember this is simply an example;
// only look at how we use the API described in the plugin's page;
// the rest of things here are of no importance

<template>
  <q-page>

    <div>
      <q-input rounded bottom-slots v-model="locationSearchTerm" label="Where to?" counter maxlength="30"
        @keydown.prevent="searchLocation">
        <template v-slot:before>
          <q-icon name="flight_takeoff" />
        </template>

        <template v-slot:append>
          <q-icon v-if="locationSearchTerm !== ''" name="close" @click="locationSearchTerm = ''"
            class="cursor-pointer" />
          <q-icon name="search" />
        </template>

      </q-input>
    </div>
    <div>
      <div @change="position">
        GPS position:

        <strong v-if="position != 'determining...'">Within {{ parseInt(position.coords.accuracy) }} meters</strong>
        <strong v-else> {{ position }} </strong>
      </div>

      <div @click.stop="getAddressFromLatLng" id="map" style="height:90vh;"></div>
    </div>
  </q-page>

</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import { Geolocation } from '@capacitor/geolocation'
import axios from 'axios'

import "leaflet/dist/leaflet.css";
import * as L from 'leaflet'

const reverseApi = axios.create({ baseURL: "https://nominatim.openstreetmap.org" })
const routesApi = axios.create({ baseURL: "https://router.project-osrm.org" })

let geoId
let mapConfig = {
  zoomControl: true,
  zoom: 1,
  zoomAnimation: false,
  fadeAnimation: true,
  markerZoomAnimation: true
}
const position = ref('determining...')
const initialMap = ref(null)
const locationSearchTerm = ref("")
const totalTime = ref(0)
const distance = ref(0)


function searchLocation() {
  console.log(locationSearchTerm.value)
}

function addMarker(latlng, label) {
  var newMarker = new L.marker(latlng)

  let distanceKm = (distance.value / 1000.0).toFixed(2)
  let travelTimeHr = (totalTime.value / 3600).toFixed(2)

  label += `</br><strong>Distance: ${distanceKm} Km</strong></br><strong>Travel Time: ${travelTimeHr} Hr</strong>`
  newMarker.addTo(initialMap.value).bindPopup(label).openPopup()
}

function getAddressFromLatLng() {

  initialMap.value.on('click', (e) => {
    console.log("to: ", e.latlng)
    let url = `/reverse?lat=${e.latlng.lat}&lon=${e.latlng.lng}&format=json&accept-language=en`
    reverseApi.get(url)
      .then((resp) => {
        let addr = resp.data.display_name
        console.log("address: ", addr)
        addMarker(e.latlng, addr)
      }).catch((err) => {
        console.log("api error: ", err)
      })
    getRoute(e.latlng)
  })
}

function getCurrentPosition() {
  let status = Geolocation.checkPermissions()
  status.then((s) => {
    console.log("permission: ", s.location)
    if (s.location !== "granted") {
      let req = Geolocation.requestPermissions()
      req.then((r) => {
        console.log("permission reqs: ", r)
      }).catch((err) => {
        console.log("error permission reqs: ", err)
      })

    }
  }).catch((err) => {
    console.log("permission error: ", err)
  })
  Geolocation.getCurrentPosition().then(newPosition => {
    console.log('Current', newPosition)
    position.value = newPosition
    initialMap.value = L.map("map", mapConfig).setView([position.value.coords.latitude, position.value.coords.longitude], 16)
    L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
      maxZoom: 19,
      attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>'
    }).addTo(initialMap.value);
  })
}

function getRoute(toLatLng) {
  let p1lng = position.value.coords.longitude
  let p1lat = position.value.coords.latitude
  let p2lng = toLatLng.lng
  let p2lat = toLatLng.lat

  routesApi.get(`/route/v1/driving/${p1lng},${p1lat};${p2lng},${p2lat}?steps=true`)
    .then((resp) => {
      distance.value = 0
      totalTime.value = 0
      const polylineList = [];

      const routes = resp.data.routes;
      routes.forEach((route) => {
        distance.value += route.distance
        totalTime.value += route.duration
        const legs = route.legs;
        legs.forEach((leg) => {
          const steps = leg.steps;
          steps.forEach((step) => {
            const polyline = step.maneuver.location.reverse();
            polylineList.push(polyline);
          });
        });
      });

      var polyline = L.polyline(polylineList, { color: 'blue' }).addTo(initialMap.value)
      initialMap.value.fitBounds(polyline.getBounds())
    })


}

function loadmap(pos) {
  console.log("pos", pos)

  let points = [pos.value.coords.latitude, pos.value.coords.longitude]
  L.marker(points).addTo(initialMap.value)

  L.circle(points, {
    color: 'red',
    fillColor: "#f03",
    fillOpacity: 0.075,
    radius: pos.value.coords.accuracy
  }).addTo(initialMap.value)

}

onMounted(() => {
  getCurrentPosition()
  console.log("position: ", position.value)
  // we start listening
  geoId = Geolocation.watchPosition({}, (newPosition, err) => {
    console.log('New GPS position: ', newPosition)
    position.value = newPosition
    loadmap(position)
  })

})

onBeforeUnmount(() => {
  // we do cleanup
  Geolocation.clearWatch(geoId)
})

</script>
