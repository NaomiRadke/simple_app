<template>
    <div>
      <!-- Font awseome stylesheet -->
      <link rel="stylesheet" href="http://maxcdn.bootstrapcdn.com/font-awesome/4.3.0/css/font-awesome.min.css"/>
    
      <div id="bg-map" class="bg-map">
        <l-map
          ref="map"
          v-model="zoom"
          v-model:zoom="zoom"
          :center="center"
          :options="{zoomControl:false, attributionControl:false}"
          crs="CRS.EPSG3857"
          :minZoom="minzoom"
          @mousedown="mouseDownHandler"
        >
    
        <!-- Leaflet control tools here: -->
        <l-control-scale position="bottomright" :imperial="false" :metric="true"></l-control-scale>  
        <l-control-layers position="topright"></l-control-layers>
      
    
        <!-- Background maps: -->
        <l-tile-layer
          v-for="tileProvider in tileProviders"
          :key="tileProvider.name"
          :name="tileProvider.name"
          :visible="tileProvider.visible"
          :url="tileProvider.url"
          :attribution="tileProvider.attribution"
          layer-type="base"
          :zIndex="baseZIndex" />
    
          <!-- Borders (region, community) from Geoserver as cached tile layers, always visible on base map -->    
        
      
    
        <!-- Mask area except region -->
        <l-geo-json 
          ref="geojsonMaske"
          :geojson="MaskeData"
          :options="maskeOptions"/>
    
        <!-- Layers for Säule 2 -->
        <div v-if="applicationState === 'tours'"> 
          <!-- <l-wms-tile-layer 
          :base-url="wmsurl"
          :layers="layer_s2"
          :format="format_tile"
          :tiled="true"
          /> -->
          <l-wms-tile-layer
            v-for="wmsLayer in getCheckedLayersS2"
            :key="wmsLayer.path"
            :layers="wmsLayer.path"
            :format="format_tile"
            :transparent="true"
            :tiled="true"
            :base-url="wmsurl"
            :visible="true" />
    
          <!-- Borders for Fokusgebiete in geojson format: -->  
          <div v-if="show_focus_regions == true ">
            <l-geo-json
            ref="geojsonNeckartal"
            :geojson="NeckartalData"
            :options="geojsonOptions"
            @click="$router.push({name: 'Neckartal_Scrolly'}); zoomToNeckartal();" />
    
            <l-geo-json 
            ref="geojsonKaltental"
            :geojson="KaltentalData"
            :options="geojsonOptions"
            @click="$router.push({name: 'Kaltental_Scrolly'}); zoomToKaltental();" />
    
          </div>
          
        </div>
    
        <!-- Setting markers or polygons for scrollytelling -->
        <l-marker v-show="scroll_active" ref="scrolly-marker" v-for="marker in scrolly_markers" :lat-lng="marker" :key="marker"> 
          <l-popup> Hello! </l-popup>
        </l-marker>
    
        <l-geo-json 
          v-if="geojson_scrolly !== null" 
          ref="geojsonScrolly"
          :geojson="geojson_scrolly"
          :options="geojsonOptions"
          @ready="zoomToPolygon()"
          />
      
    
        </l-map>
      </div>
    </div>
    </template>
    
    <script>
    // import libraries + styles
    import {
      LMap,
      LTileLayer,
      LControlZoom,
      LIcon,
      LMarker,
      LTooltip,
      LPopup,
      LGeoJson,
      LWmsTileLayer,
      LControlLayers,
      LControlScale,
      LControl,
    } from "@vue-leaflet/vue-leaflet";
    import "leaflet/dist/leaflet.css";
    import '../assets/css/customLeafletControls.css';
    import "leaflet.zoomhome";
    import "leaflet.zoomhome/src/css/leaflet.zoomhome.css";
    import {LControlZoomBox} from 'leaflet-zoombox';
    import "leaflet-zoombox/L.Control.ZoomBox.css";
    import {mapGetters, mapActions} from "vuex";
    import "leaflet-side-by-side";
    import { CRS } from "leaflet";
    
    // import components:
    import OpacitySlider from "@/components/OpacitySlider.vue";
    import GetFeatureContent from '@/components/PopupContent.vue';
    import axios from 'axios';
    
    // import border geojsons:
    import KreisData from "../assets/geojson/kreisgebiete.json";
    import RegionData from "../assets/geojson/regionsgebiet.json";
    import GemeindeData from "../assets/geojson/gemeindegebiete.json";
    import MaskeData from "../assets/geojson/maske.json";
    import KaltentalData from "../assets/geojson/kaltental_grenze.json";
    import NeckartalData from "../assets/geojson/neckartal_grenze.json";
    
    export default {
      name: 'Map',
      components: {
        LMap,
        LTileLayer,
        LControlZoom,
        LIcon,
        LMarker,
        LTooltip,
        LPopup,
        LGeoJson,
        LWmsTileLayer,
        LControlLayers,
        LControlScale,
        LControl,
        OpacitySlider,
        LControlZoomBox,
        mapGetters,
        mapActions,
        GetFeatureContent, 
        CRS
      },
      data() {
        return {
          // center: [47.313220, -1.319482],
          // map settings
          zoom: 10,
          minzoom: 10,
          center: [48.7667, 9.1833],
    
          // background map
          tileProviders: [
            {
              name: 'OpenStreetMap',
              visible: true,
              attribution:
                '&copy; <a target="_blank" href="http://osm.org/copyright">OpenStreetMap</a> contributors',
              url: 'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',
            },
            {
              name: 'Stadia.AlidadeSmooth',
              visible: false,
              url: 'https://tiles.stadiamaps.com/tiles/alidade_smooth/{z}/{x}/{y}{r}.png',
              attribution:
                '&copy; <a href="https://stadiamaps.com/">Stadia Maps</a>, &copy; <a href="https://openmaptiles.org/">OpenMapTiles</a> &copy; <a href="http://openstreetmap.org">OpenStreetMap</a> contributors',
            },
            {
              name: 'ESRI.WorldTopoMap',
              visible: false,
              url: 'https://server.arcgisonline.com/ArcGIS/rest/services/World_Topo_Map/MapServer/tile/{z}/{y}/{x}',
              attribution:
                'Tiles &copy; Esri &mdash; Esri, DeLorme, NAVTEQ, TomTom, Intermap, iPC, USGS, FAO, NPS, NRCAN, GeoBase, Kadaster NL, Ordnance Survey, Esri Japan, METI, Esri China (Hong Kong), and the GIS User Community',
            },
          ],
    
          // tiled wms layers
          wmsurl:'http://'+process.env.VUE_APP_EC2_IP+'/api/geoserver/gwc/service/wms',
          format_tile: "image/png",
          baseZIndex: 1,
          layer_s2: null,
    
          // border polygons (geojson)
          geojsonOptions: {
            style: {fillColor: '#8fbcf2',
                    color: '#325ea8',
                    radius: 4}
          },
          gemeindeOptions: {
            style: {color: '#8f8888',
                    weight: 1.5,
                    fillOpacity: 0,
                    zIndex: 500}
          },
          regionOptions: {
            style: {color: '#8f8888',
                    weight: 3,
                    fillOpacity:0,
                    zIndex: 400}
          },
          kreisOptions: {
            style: {color: '#8f8888',
                    weight: 3,
                    fillOpacity:0,
                    zIndex: 401}
          },
          maskeOptions: {
            style: {fillColor: '#9e9d9b',
                    fillOpacity: 0.5,
                    weight: 0}
          },
          geojson: null,
          show_focus_regions: false,
          RegionData: RegionData,
          KreisData: KreisData,
          GemeindeData: GemeindeData,
          MaskeData: MaskeData,
          KaltentalData: KaltentalData,
          NeckartalData: NeckartalData,
    
          // scrolly markers and polygons
          marker: [0,0],
          scrolly_markers: [[0,0]],
          scroll_active: false,
          geojson_scrolly: null,
          markerOptions: {opacity: 0, width:"1px", height:"1px"},
    
          // popups on layer click
          popupOptions: {
            maxWidth: 400
          },
    
          // slider
          sideBySide: null,   
        };
      },
    
      async created () {
        // get Fokusräume geojson data after event was emitted
        this.emitter.on("show-focus-regions", e => {
          this.showFocusRegions(e);
        }),
        this.emitter.on("hide-focus-regions", e => {
          this.show_focus_regions == false;
        })
        // this.getBorderPolygons(); 
      },
    
      async beforeMount() {
        // HERE is where to load Leaflet components!
        this.mapIsReady = true;
    
        // Options for the Geojson Polygons
        this.geojsonOptions.onEachFeature = (feature, layer) => {
          var self = this;
          layer.on('mouseover', function() {
            layer.setStyle({fillColor: '#b1d2f0', fillOpacity: 0.5, weight:3})
          });
          layer.on('mouseout', function() {
            layer.setStyle({fillOpacity: 0, color:'#325ea8', weight:3})
          });
        }
        this.gemeindeOptions.onEachFeature = (feature, layer) => {
          var self = this;
          layer.on('mouseover', function() {
            layer.setStyle({color: '#f55d42', weight: 3})
          });
          layer.on('mouseout', function() {
            layer.setStyle({fillOpacity:0, color:'#8f8888'})
          });
          layer.bindTooltip(
            "<b>Gemeinde:</b> " + feature.properties.GEMEINDE + "<br/>" +
            "<b>Kreis:</b> " + feature.properties.KREIS + "<br/>", { permanent: false, sticky: true });
          
         layer.on("click", function (event) {
            var map = self.$refs.map.leafletObject
            var bbox = layer.getBounds()
            map.flyToBounds(bbox, {padding: [250,100], maxZoom:12});
          })
        }
      },
        
      mounted() { 
        // listening for events
    
        this.emitter.on("add-split", e => {
          this.addSplit();
        });
        this.emitter.on("remove-split", e => {
          this.removeSplit();
        });
        this.emitter.on('unmount-split', e => {
          this.removeSplit();
        });
        this.emitter.on("goto-nth-position", e => {
          this.gotoNthPosition(e);
        });
        this.emitter.on("goto-nth-polygon", e => {
          if(this.geojson_scrolly === e) {
            this.zoomToPolygon();
          } else {
            this.gotoNthPolygon(e);
          }
        });
        this.emitter.on("show-layer-s2", e => {
          this.layer_s2 = "isap:schaden"
        })
    
        // add some map components
        this.addHomeButton();
        this.addZoomBox();
    
      },
      beforeUnmount() {
      },
      methods: {
    
        gotoNthPosition(e) {    
          this.scrolly_markers.push(e);
          var meanLat=e[0];
          var meanLng=e[1];
          const map = this.$refs.map.leafletObject;
          this.center = [meanLat, meanLng]
          this.zoom = 13;
          // map.setView({lat: meanLat, lng: meanLng}, 11);
        },
    
        gotoNthPolygon(e) {
          this.geojson_scrolly = e;
        },
    
        zoomToPolygon() {
          var bbox = this.$refs.geojsonScrolly.leafletObject.getBounds();
          var map = this.$refs.map.leafletObject;
          map.flyToBounds(bbox, {padding: [300,300]});
        },
    
        addHomeButton() {
          (async() => {
            while(typeof this.$refs.map.leafletObject.options === 'undefined') {
              await new Promise(resolve => setTimeout(resolve, 100));
            }
            const map = this.$refs.map.leafletObject;
            const zoomHome = L.Control.zoomHome({position: 'topright', zoomHomeIcon: "home"});
            zoomHome.addTo(map);   
          })();
        },
        
        addZoomBox() {
          (async() => {
            while(typeof this.$refs.map.leafletObject.options === 'undefined') {
              await new Promise(resolve => setTimeout(resolve, 100));
            }
            const boxoptions = {
            modal: true,
            title: "Box area zoom",
            position: "topright"
            };
            const boxcontrol = L.control.zoomBox(boxoptions);
            const map = this.$refs.map.leafletObject;
            map.addControl(boxcontrol); 
          })();
        },
    
        addSplit() {
          // Initialize variables
          const map = this.$refs.map.leafletObject;
          const layer_split_1 = this.getCheckedLayers[0].path;
          const layer_split_2 = this.getCheckedLayers[1].path;
          var leaflet_layer_1 = null; 
          var leaflet_layer_2 = null;
          var layers = [];
          // First set activeSplit to false for all layers, then value of split layers to "true"
          this.getCheckedLayers.forEach((item) => {item.activeSplit = false});
          this.getCheckedLayers[0].activeSplit = true;
          this.getCheckedLayers[1].activeSplit = true;
          // Extract layer objects of WMS tile layers
          map.eachLayer(function(layer) {
            if (layer.wmsParams != null) {
              if (layer.wmsParams.layers == layer_split_1) {
              leaflet_layer_1 = layer
              }
              if (layer.wmsParams.layers == layer_split_2) {
              leaflet_layer_2 = layer
              }
            }
          layers.push(layer); 
          });
      
          // Add slider
          if (this.sideBySide == null){
            this.sideBySide = L.control.sideBySide(leaflet_layer_1, leaflet_layer_2);
            this.sideBySide.addTo(map);
            map.dragging.disable()
          }
        },
    
        removeSplit() {
          if (this.sideBySide !== null) {
            const map = this.$refs.map.leafletObject;
            map.removeControl(this.sideBySide);
            // Set  values of split layers to "false"
            this.getCheckedLayers[0].activeSplit = false;
            this.getCheckedLayers[1].activeSplit = false;
            this.sideBySide = null;
            map.dragging.enable()
          }
        },
    
        addMarker(e) {
          // check if activeTooltip prop is true for any layer
          if((this.getCheckedLayers.some( object => object.activeTooltips === true))) {
          // set marker at latlng position
          this.marker = e.latlng;
          // open popup on marker with a delay of 100 ms
          setTimeout(() => 
            this.$refs.marker.leafletObject.openPopup(), 100);
          
          // call GetFeatureInfo method in child component (PopupContent)
          const map = this.$refs.map.leafletObject
          this.$refs.popupContent.getFeatureInfo(this.marker, map);
          }
        },
    
        mouseDownHandler() {
          this.$refs.map.leafletObject.addEventListener('mouseup', this.mouseupHandler);
          this.$refs.map.leafletObject.addEventListener('drag', this.dragHandler);
        },
    
        mouseupHandler(e){
          this.addMarker(e);
          this.cleanUpEventListeners();
        },
    
        dragHandler(){
          this.cleanUpEventListeners();
        },
    
        cleanUpEventListeners(){
          this.$refs.map.leafletObject.removeEventListener('mouseup', this.mouseupHandler);
          this.$refs.map.leafletObject.removeEventListener('drag', this.dragHandler)
        },
      
        showFocusRegions(e) {
          this.show_focus_regions = true;
          var map = this.$refs.map.leafletObject
          this.zoom = 11
        },
    
        zoomToNeckartal() {
           // get the leaflet map object
          var map = this.$refs.map.leafletObject
          // shift the map view to the right depending on the current container width
          var offset_factor = 1-(parseFloat(this.getContainerWidth)/100)  
          var bbox = this.$refs.geojsonNeckartal.leafletObject.getBounds()    
          map.fitBounds(bbox, {padding:[300,0]}) 
          this.center = [48.7705, 9.2413]
          // map.panBy([-150, 0]) 
          },
    
        zoomToKaltental() {
          // get the leaflet map object
          var map = this.$refs.map.leafletObject
          var bbox = this.$refs.geojsonKaltental.leafletObject.getBounds()
          map.fitBounds(bbox, {padding:[300,0]})
          this.center = [48.758, 9.136310]
          // map.panBy([-x_offset,0]) // pan x pixels to the right
        },
    
        getBorderPolygons () {
            // Get Kreis polygon as geojson 
            // Define the API URL
            const Kreis_url = 'http://'+process.env.VUE_APP_EC2_IP+'/api/geoserver/isap/ows?service=WFS&version=1.0.0&request=GetFeature&typeName=isap%3Akreisgebiete_2018_4326&maxFeatures=200&outputFormat=application%2Fjson';
    
            // Next, make the get request
            axios.get(Kreis_url)
            .then((response) => {
                this.KreisData = response.data;
            })
            .catch(error => {
                console.log(error);
            });
    
            // Get Gemeinde polygon as geojson 
            // Define the API URL
            const Gemeinde_url = 'http://'+process.env.VUE_APP_EC2_IP+'/api/geoserver/isap/ows?service=WFS&version=1.0.0&request=GetFeature&typeName=isap%3Agemeindegebiete_2018_4326&maxFeatures=200&outputFormat=application%2Fjson';
    
            // Next, make the get request
            axios.get(Gemeinde_url)
            .then((response) => {
                this.GemeindeData = response.data;
            })
            .catch(error => {
                console.log(error);
            });
    
            // Get Region polygon as geojson 
            // Define the API URL
            const Region_url = 'http://'+process.env.VUE_APP_EC2_IP+'/api/geoserver/isap/ows?service=WFS&version=1.0.0&request=GetFeature&typeName=isap%3Aregionsgebiet_2018_4326&maxFeatures=200&outputFormat=application%2Fjson';
    
            // Next, make the get request
            axios.get(Region_url)
            .then((response) => {
                this.RegionData = response.data;
            })
            .catch(error => {
                console.log(error);
            });
          },
    
        zoomToGemeinde() {
          // get the leaflet map object
          var map = this.$refs.map.leafletObject
          // shift the map view to the right depending on the current container width
          var center_point = map.getSize().divideBy(2)
          var offset_factor = parseFloat(this.getContainerWidth)/100
          var x_offset = center_point.x*offset_factor
          var bbox = this.$refs.geojsonKalt.leafletObject.getBounds()
          map.fitBounds(bbox)
          map.panBy([-x_offset,0]) // pan x pixels to the right
    
        }
      },
      
    };
    </script>
    
    <style scoped>
    .bg-map {
      position: fixed;
      height: 100vh; 
      width: 100vw;
      margin: 0;
      z-index: 0;
    };
    :root {
      --toggle-bg-on: red;
    };
    </style>