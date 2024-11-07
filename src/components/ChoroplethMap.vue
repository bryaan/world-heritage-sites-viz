<script setup>
import { ref, reactive, onMounted, computed } from 'vue'
import Plotly from 'plotly.js-dist'
import _ from 'lodash'
import iso from 'iso-3166-1'

import whcCsv from '../assets/whc-sites.csv'
// console.log(whcCsv)

// Splits a comma separted string into an array of strings, and
// where there are no commas outputs a single element list.
function splitByComma(str) {
    // Check if the string contains a comma
    if (str.includes(',')) {
        // Split by comma and trim each element
        return str.split(',').map(s => s.trim());
    } else {
        // Return the string as a single-element array
        return [str.trim()];
    }
}

// Get the iso codes available in dataset as a list of lists of strings.
// Where there is one iso code a single element list will be returned.
// Where there are multiple comma separated iso codes a list of strings will be returned.
const isoCodes = whcCsv.map((item) => splitByComma(item.iso_code))

// Get a list of strings where there is an element per isocode 
const flattenedIsoCodes = isoCodes.flat();

// Get an object with keys being the country isocode and values being how many times it appears.
const numOfSitesByCountry = _.countBy(flattenedIsoCodes);

// Remove the element with an empty string as the key
// Jerusalem is the element.
delete numOfSitesByCountry[""];

// Get the iso2 code keys as a list
const iso2CodesList = Object.keys(numOfSitesByCountry);

// Need to convert ISO-2 codes to ISO-3 for use with plotly choropleth map
const iso3CodesList = iso2CodesList.map((iso2code) => iso.whereAlpha2(iso2code).alpha3)

// Get country names
const countryNameList = iso2CodesList.map((iso2code) => iso.whereAlpha2(iso2code).country)

// Get the number of sites per country as a list
const numOfSitesByCountryList = Object.values(numOfSitesByCountry);


// Create short descriptions info array for all sites in a country 
const createDescriptionsObject = () => {
    const descriptionsObject = {}
    whcCsv.map((item) => {
        const locationIsoCodes = splitByComma(item.iso_code)
        locationIsoCodes.forEach((iso2code) => {
            const isoInfo = iso.whereAlpha2(iso2code)
            if (isoInfo) {
                const iso3Code = isoInfo.alpha3
                descriptionsObject[iso3Code] = descriptionsObject[iso3Code] || [];
                descriptionsObject[iso3Code].push({
                    country: isoInfo.country,
                    category: item.category,
                    dangerLevel: item.danger,
                    dateInscribed: item.date_inscribed,
                    shortDescription: item.short_description_en
                })
            }
        })
    })
    return descriptionsObject
}

const descriptionsObject = createDescriptionsObject()

// Ref for the map div
const mapContainer = ref(null);

const renderMap = () => {
    const data = [
        {
            type: 'choropleth',
            locationmode: 'ISO-3',
            locations: iso3CodesList,
            z: numOfSitesByCountryList,
            text: countryNameList,
            colorscale: 'Viridis',
            colorbar: { title: 'Number of Sites' },
        },
    ];

    const layout = {
        title: '',
        geo: {
            projection: {
                // https://stackoverflow.com/questions/32041078/plotly-map-projection-types
                type: 'natural earth',
            },
        },
    };

    Plotly.newPlot(mapContainer.value, data, layout, { responsive: true });
};

// Current selected country description info
const selectedCountryInfo = ref([])

// Current index for pagination
const currentIndex = ref(0);

// Computed property for the current item being displayed
const currentItem = computed(() => {
    return selectedCountryInfo.value[currentIndex.value] || {};
});

// Methods for pagination
function prevItem() {
    if (currentIndex.value > 0) {
        currentIndex.value--;
    }
}

function nextItem() {
    if (currentIndex.value < selectedCountryInfo.value.length - 1) {
        currentIndex.value++;
    }
}

onMounted(() => {
    renderMap();

    if (mapContainer.value) {
        // Register event for clicking a country on the map
        mapContainer.value.on('plotly_click', function (data) {
            const iso3Code = data.points[0].location
            selectedCountryInfo.value = descriptionsObject[iso3Code]
            currentIndex.value = 0
        });
    }
});
</script>


<template>
    <div class="container">
        <div class="page-heading">Number of World Heritage Sites by Country</div>

        <div class="map-container">
            <div ref="mapContainer" style="width: 100%; height: 500px;"></div>
        </div>

        <h2>World Hertitage Sites For Selected Country Details</h2>
        <div v-if="selectedCountryInfo.length == 0">
            Please select a country to view details.
        </div>

        <div class="site-info-container" v-if="selectedCountryInfo.length > 0">

            <div class="pagination-controls">
                <button @click="prevItem" :disabled="currentIndex === 0">Previous</button>
                <button @click="nextItem" :disabled="currentIndex === selectedCountryInfo.length - 1">Next</button>
            </div>

            <div class="site-info">
                <div>
                    <label>Country: </label>
                    <span>{{ currentItem.country }}</span>
                </div>
                <div>
                    <label>Category: </label>
                    <span>{{ currentItem.category }}</span>
                </div>
                <div>
                    <label>Danger Level: </label>
                    <span>{{ currentItem.dangerLevel }}</span>
                </div>
                <div>
                    <label>Date Inscribed: </label>
                    <span>{{ currentItem.dateInscribed }}</span>
                </div>
                <div>
                    <span v-html="currentItem.shortDescription"></span>
                </div>
            </div>
        </div>

    </div>
</template>

<style scoped>
.container {
    width: 100vw;
    padding-bottom: 50vh;
}

.page-heading {
    padding-top: 16px;
    font-size: 2rem;
    font-weight: bold;
}

.map-container {
    width: 100%;
}

.site-info-container {
    width: 500px;
    margin: 0 auto;
}

.site-info {
    padding: 4px 0;
    text-align: left;
}

.site-info label {
    font-weight: bold;
}

.pagination-controls {
    margin: 20px 0;
}

.pagination-controls button {
    margin: 0 4px;
    padding: 10px 20px;
}
</style>