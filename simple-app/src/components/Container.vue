<template>
    <div id="header"> 
        <h1>..__/\/^- earthquake watch -^\/\__.. </h1>
    </div>
    <div id="container-wrapper" :class="{show: doShow}"> <!--:style="{width: getContainerWidth}"-->
        <div id="content-body"> 
          <p> This map visualizes earth quake data of the past day across the world. 
              The data is updated every minute and sourced from the US government's <a href="https://earthquake.usgs.gov/earthquakes/feed/v1.0/geojson.php">earthquake API</a>.
          </p>
          <p> The individual circle size is based on the earthquake's magnitude.</p>
          <eqInfo />
        </div> 

    </div>
    <div id="handle" :class="{show: doShow}" @click="showhide">
      <span v-html="arrows"></span>
  </div>
</template>

<script>
import eqInfo from "@/views/eqInfo.vue";

export default {
    name: 'app-container',
    components: { eqInfo },
    props: {geojson: Object},
    data: function () {
        return {
            doShow: true,
            arrows: "&laquo;"
        }
    },
    methods: {
        showhide () {
            this.doShow = !this.doShow;
            if (this.doShow) this.arrows = "&laquo;";
            else this.arrows = "&raquo;";
        }
    },
    computed: {
     
    },
}
</script>

<style scoped>
#container-wrapper {
  position: absolute;
  top: 80px;
  bottom: 30px;
  left: -33%;
  transition: left 0.1s ease-in;
  z-index: 80;

  width: 33%;
  overflow: auto;

  background-color: white;
  border: 1px solid var(--accent);
  border-radius: 0 10px 10px 0;
  opacity: 0.8;
  padding: 0px;
  box-shadow: 0 4px 8px 0 rgba(0, 0, 0, 0.2), 0 6px 20px 0 rgba(0, 0, 0, 0.19);
}


#container-wrapper.show {
  left: 0px;
  transition: left 0.1s ease-in;
}

#handle {
  cursor: pointer;
  background-color: white;
  border-top: 1px solid #aaa;
  border-right: 1px solid #aaa;
  border-bottom: 1px solid #aaa;
  border-radius: 0 7px 7px 0;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.5);
  width: 15px;
  height: 40px;
  line-height: 38px;
  padding: 0 0 0 2px;

  position: fixed;
  top: 50%;
  left: 0%;
  z-index: 70;
  transition: left 0.1s ease-in;
}

#handle:hover {
  background-color: rgba(31, 221, 148, 0.925);
}

#handle.show {
  left: 33%;
  transition: left 0.1s ease-in;
}

#header {
  position: fixed;
  background-color: white;
  border: 1px solid white;
  border-radius: 0px;
  opacity: 0.8;
  text-align: center;
  width: 33%;
  height: 55px;
  left: 0px;
  top: 15px;
  

  font-size: 0.9vw;

  /* box-shadow: 0 1px 2px 0 rgba(0, 0, 0, 0.2), 0 3px 10px 0 rgba(0, 0, 0, 0.19); */
}

#content-body{
    overflow: auto;
    padding: 10px 50px 10px 50px;
}
p{
  font-size: 0.9vw;
}



</style>