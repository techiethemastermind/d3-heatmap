<template>
    <div class="chart-wrap">
        <h1>{{ title }}</h1>
        <div id="chart">
            <svg></svg>
        </div>
    </div>
</template>

<script>

    import * as d3 from 'd3';

    export default {
        name: 'HeatChart',
        props: {
            title: String,
            chartData: Array
        },
        watch: {
            chartData(val) {
                if (val) {
                    this.renderChart(val);
                }
            }
        },
        data() {
            return {
                chart: null
            };  
        },
        mounted() {
            if (this.chartData.chartData.length > 0) {
                this.renderChart(this.chartData);
            }
        },
        methods: {
            renderChart(val) {
                const chartData = val.chartData;
                const margin = {top: 30, right: 90, bottom: 100, left: 90},
                      width = 1100 - margin.left - margin.right,
                      height = 480 - margin.top - margin.bottom;
                const timeFormat = d3.timeFormat('%H:%M');
                const rows = 30;
                const cols = 150;

                const svg = d3.select('svg')
                    .attr('width', width + margin.left + margin.right)
                    .attr('height', height + margin.top + margin.bottom)
                    .append('g')
                    .attr('transform', 'translate(' + margin.left + ',' + margin.top + ')');

                let xMin = d3.min(chartData, function(d) { return Math.min(d.time); });
                let xMax = d3.max(chartData, function(d) { return Math.max(d.time); });
                let yMin = d3.min(chartData, function(d) { return Math.min(d.value); });
                let yMax = d3.max(chartData, function(d) { return Math.max(d.value); });
                let xCell = Math.round((xMax - xMin) / (cols - 1));
                let yCell = Math.round((yMax - yMin) / (rows - 1));

                let xRange = [];
                let yRange = [];

                for (let i = 0; i < cols; i++) {
                    xRange.push(xMin + i * xCell);
                }

                for (let j = 0; j < rows; j++) {
                    yRange.push(yMin + j * yCell);
                }

                // Build X axis
                xMax = xRange[xRange.length - 1] + xCell;
                const xScale = d3.scaleLinear()
                    .domain([xMin, xMax])
                    .range([0, width]);
                const xAxis = d3.axisBottom(xScale)
                    .tickFormat(d => {
                        return timeFormat(d * 1000);
                    })
                svg.append("g")
                    .attr("transform", "translate(0," + height + ")")
                    .call(xAxis);

                // Build Y axis
                yMax = yRange[yRange.length - 1] + yCell;
                const yScale = d3.scaleLinear()
                    .domain([yMin, yMax])
                    .range([height, 0]);
                const yAxis = d3.axisLeft(yScale);
                svg.append("g")
                    .call(yAxis);
            
                // Build Heatmap Data
                let dataList = [];

                xRange.forEach(x => {
                    let group = chartData.filter(d => {
                        return d.time >= x && d.time < (x + xCell);
                    });

                    for (let i = 0; i < yRange.length; i++) {

                        let data = {
                            time: x,
                            value: yRange[i],
                            count: 0
                        };

                        group.forEach(item => {
                            if (item.value >= yRange[i] && item.value < yRange[i + 1]) {
                                data.count++;
                            }
                        });

                        dataList.push(data);
                    };
                });

                const cellHeight = height / rows;
                const cellWidth = width / cols - 1;
                const colors = ['#ADD8E6', '#87CEFA', '#9CB598', '#C8CEAD', '#E6E6BA', '#E8D499', '#E2B07F', '#E67F5F', '#C55562', '#A53A66'];

                var colorScale = d3.scaleQuantile()
                    .domain([d3.min(dataList, function (d) { return d.count; }), d3.max(dataList, function (d) { return d.count; })])
                    .range(colors);

                svg.append('g')
                    .attr('class', 'chart')
                    .selectAll('.rect')
                    .data(dataList)
                    .enter().append("rect")
                    .attr("class", "rect")
                    .attr("width", cellWidth)
                    .attr("height", cellHeight)
                    .attr("x", function(d) { return xScale(d.time) + 1; })
                    .attr("y", function(d) { return yScale(d.value) - 12; })
                    .attr("fill", function(d) { 

                        if (d.count > 0) {
                            return colorScale(d.count);
                        } else {
                            return '#FAFAFA';
                        }
                        
                    })

                //legend
                let legend = svg.selectAll(".legend")
                    .data([0].concat(colorScale.quantiles()), function(d) { return d; });

                legend.enter().append("g")
                    .attr("class", "legend")
                    .append("rect")
                    .attr("x", function(d, i) { return (width/2) + (width/20) * i; })
                    .attr("y", height + 32)
                    .attr("width", (width/20))
                    .attr("height", 30 / 2)
                    .style("fill", function(d, i) { return colors[i]; })
                    
                legend.enter().append("g")
                    .style('font-size', '10')
                    .style('font-family', 'sans-serif')
                    .append("text")
                    .attr("class", "mono")
                    .text(function(d) { return (Math.round(d + 1)); })
                    .attr("x", function(d, i) { return  (width/2) + (width/20) * i; })
                    .attr("y", height + 62)
                    .attr("fill", "currentColor");

                // Brushing
                let start = [];
                let end = [];

                svg.call( d3.brush()
                    .extent( [ [0,0], [width, height] ] )
                    .on('start end', handleBrushing)
                )

                function handleBrushing(d) {
                    if (d.type == 'start') {
                        start = d.selection[0]
                    }

                    if (d.type == 'end') {
                        end = d.selection ? d.selection[1] : start;

                        if ((start == end)) {
                            return false;
                        }

                        // Get info from selection
                        let time_1 = Math.round(xScale.invert(start[0]));
                        let time_2 = Math.round(xScale.invert(end[0]));

                        let value_1 = Math.round(yScale.invert(start[1]));
                        let value_2 = Math.round(yScale.invert(end[1]));

                        let startTime = time_1 < time_2 ? time_1 : time_2;
                        let endTime = time_1 > time_2 ? time_1 : time_2;
                        let lowerValue = value_1 < value_2 ? value_1 : value_2;
                        let upperValue = value_1 > value_2 ? value_1 : value_2;

                        let sendData = {
                            startTime : startTime,
                            endTime: endTime,
                            upper: upperValue,
                            lower: lowerValue
                        }

                        console.log(sendData);
                    }
                }
                
            }
        }
    }
</script>

<style scoped>
    #chart {
        display: inline-block;
        margin-bottom: 30px;
        padding: 15px;
        border-radius: 5px;
    }
    svg {
        width: 1100px;
        height: 480px;
    }
</style>
