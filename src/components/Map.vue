<template>
    <div class="container">
        <div class="col-md-12" style="margin:40px">
            <h2>Near by Restaurant</h2>
        </div>
        <div class="row">
            <div class="col-md-8">
                <GmapMap
                    :center="myCoordinates"
                    :zoom="zoom"
                    style="width:100%; height:720px; auto;"
                    ref="mapRef"
                    @dragend="handleDrag"
                >
                  <GmapMarker
                        :key="index"
                        v-for="(m, index) in locations"
                        :position="m.position"
                        :clickable="true"
                        :draggable="false"
                        @click="toggleInfoWindow(m,index)"
                    />

                    <GmapInfoWindow
                        :options="infoOptions"
                        :position="infoWindowPos"
                        :opened="infoWinOpen"
                        @closeclick="infoWinOpen=false"
                    >
                        <div v-html="infoContent"></div>
                    </GmapInfoWindow>
                </GmapMap>
            </div>
            <div class="col-md-4">
                <b-input-group class="mt-3">
                    <b-input type="text" placeholder="Enter place" v-model="searchplace"></b-input>
                    <b-button variant="info" @click="search">Search</b-button>
                </b-input-group>
                <b-table striped hover sticky-header="650px" :items="items"></b-table>
            </div>
        </div>
    </div>
</template>
<script>
import Axios from "axios";

    export default {
        data() {
            return {
                map: null,
                myCoordinates: {
                    lat: 13.828253,
                    lng: 100.528451
                },
                zoom: 7,
                items: [
                { number: 0, name: '' }
                ],
                locations : [],
                searchplace: '',

                infoContent: '',
                infoWindowPos: {
                lat: 0,
                lng: 0
                },
                infoWinOpen: false,
                currentMidx: null,
                //optional: offset infowindow so it visually sits nicely on top of our marker
                infoOptions: {
                pixelOffset: {
                    width: 0,
                    height: -35
                }
                },
            }
        },
        created() {
            // does the user have a saved center? use it instead of the default
            if(localStorage.center) {
                this.myCoordinates = JSON.parse(localStorage.center);
            } else {
                // get user's coordinates from browser request
                this.$getLocation({})
                    .then(coordinates => {
                        this.myCoordinates = coordinates;
                    })
                    .catch(error => alert(error));
            }

            // does the user have a saved zoom? use it instead of the default
            if(localStorage.zoom) {
                this.zoom = parseInt(localStorage.zoom);
            }
        },
        mounted() {
            // add the map to a data object
            this.$refs.mapRef.$mapPromise.then(map => this.map = map);
        },
        methods: {
            toggleInfoWindow (marker, idx) {
                this.infoWindowPos = marker.position;
                this.infoContent = this.getInfoWindowContent(marker);

                //check if its the same marker that was selected if yes toggle
                if (this.currentMidx == idx) {
                this.infoWinOpen = !this.infoWinOpen;
                }
                //if different marker set infowindow to open and reset current marker index
                else {
                this.infoWinOpen = true;
                this.currentMidx = idx;
                }
            },
            handleDrag() {
                // get center and zoom level, store in localstorage
                let center = {
                    lat: this.map.getCenter().lat(),
                    lng: this.map.getCenter().lng()
                };
                let zoom = this.map.getZoom();

                localStorage.center = JSON.stringify(center);
                localStorage.zoom = zoom;
            },
            getInfoWindowContent: function (marker) {
                return (`<div>
                        <h3>${marker.name}</h3>
                </div>`);
                },
            async search() {
                let place = this.searchplace ? this.searchplace : 'Bang%20sue';
                await Axios.post("http://localhost:52358/api/Map/FindNearbyPlaceByLocation", null, { params : { place } })
                    .then(response => { 
                        console.log(response);
                    if(response.status === 200){
                        console.log(response);
                        if(response.data !== null){
                        let result = response.data.results.map((data,i) => (
                            {
                                number: i + 1,
                                name: data.name
                            }
                        ));

                        let resLocations = response.data.results.map((data,i) => (
                            {
                                key: i,
                                name: data.name,
                                position: data.geometry.location
                            }
                        ));

                        this.myCoordinates.lat = parseFloat(response.data.center.lat.toFixed(4));
                        this.myCoordinates.lng = parseFloat(response.data.center.lng.toFixed(4));
                        console.log(response.data.center);
                        this.items = result;
                        this.locations = resLocations;
                        }
                    } else{
                        console.log("status : " + response.status);
                    }   
                    })
                    .catch(err => console.log(err));
            },
            async clear() {
                this.locations = [];
            }
        },
        computed: {
            mapCoordinates() {
                if(!this.map) {
                    return {
                        lat: 0,
                        lng: 0
                    };
                }

                return {
                    lat: this.map.getCenter().lat().toFixed(4),
                    lng: this.map.getCenter().lng().toFixed(4)
                }
            }
        }
    }
</script>