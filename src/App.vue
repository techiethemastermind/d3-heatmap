<template>
    <div id="app" class="app-wrap">
        <HeatChart title="HeatMap Chart" :chartData={chartData} />
        <button class="btn" @click="getChartData()">Run Query</button>
    </div>
</template>

<script>
import axios from "axios";
import HeatChart from './components/HeatChart.vue'

export default {
    name: 'app',
    components: {
        HeatChart
    },
    data() {
        return {
            chartData: []
        }
    },
    methods: {
        getChartData() {
            axios.get('https://dev.mooserocket.dk/heatmap/durationMs')
                .then(response => {
                    this.chartData = response.data;
                })
        }
    },
    mounted() {
        // this.getChartData();
    },
}

</script>

<style scoped>
    .app-wrap {
        text-align: center;
        font-family: sans-serif;
    }
</style>