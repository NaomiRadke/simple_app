<template>
    <div class="eq-info">
        <h3> Earthquake data </h3>
    
        <br />
        <p> Find a description of the earthquake parameters <a href="https://earthquake.usgs.gov/data/comcat/index.php">here.</a></p>

    </div>
</template>

<script>

    export default {
        name: 'eqInfo',
        components: {
           
        },
        data() {
            return {
                data_clean: {}
            }
        },
        updated () {
            this.cleanData();
        },
 
        computed: {
          

        },
        methods: {
            cleanData() {
                this.data_clean = this.getCurrData;  
                this.data_clean.time = Date(this.getCurrData.time);  // turn string format to date format
                this.data_clean.updated = Date(this.getCurrData.updated);
            }  
        },
    }
</script>

<style scoped>
h3 {
    text-align: center;
}
.eq-info {
    border: 2px solid rgb(129, 87, 117);
    border-radius: 5px;
    overflow-y: auto;
}
p {
    margin: 5px
}
li {
    font-size: 12px;
}
#info {
    text-align: center;
}

</style>