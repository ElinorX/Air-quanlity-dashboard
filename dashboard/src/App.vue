<template>
  <div id="app" class="container">
    <!-- Title area -->
    <header class="header">
      <h1>Air Quality Monitoring Data Visualization</h1>
      <p>{{ timeRange }}</p>
    </header>

    <!-- First row: Time series chart -->
    <div class="row">
      <div class="card">
        <h2>Pollutant Concentration Trends</h2>
        <div class="pollutant-selector">
          <label>Select Pollutant:</label>
          <select v-model="selectedPollutant">
            <option v-for="p in metadata.pollutants" :key="p" :value="p">
              {{ p }}
            </option>
          </select>
        </div>
        <div ref="timeSeriesChart" class="chart"></div>
      </div>
    </div>

    <!-- Second row: Geographic distribution map -->
    <div class="row">
      <div class="card">
        <h2>Monitoring Station Distribution</h2>
        <div class="station-selector">
          <select v-model="selectedStation" @change="handleStationChange">
            <option value="">All Stations</option>
            <option v-for="station in stations" :key="station['Site ID']" :value="station['Site ID']">
              {{ station['Local Site Name'] }}
            </option>
          </select>
        </div>
        <div ref="mapChart" class="chart"></div>
      </div>
    </div>

    <!-- Third row: Statistical charts -->
    <div class="row two-charts">
      <div class="card">
        <h2>Station Pollutant Coverage</h2>
        <div ref="barChart" class="chart"></div>
      </div>
      <div class="card">
        <h2>AQI Distribution</h2>
        <div ref="pieChart" class="chart"></div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, watch } from 'vue'
import * as d3 from 'd3'

export default {
  name: 'App',
  setup() {
    const timeSeriesChart = ref(null)
    const mapChart = ref(null)
    const barChart = ref(null)
    const pieChart = ref(null)
    const selectedPollutant = ref('')
    const selectedStation = ref('')
    
  const data = ref(null)
    const stations = ref([])
    const metadata = ref({ pollutants: [] })
    const timeRange = ref('')

    const loadData = async () => {
      try {
        const response = await fetch('/air_quality_visualization.json')
        data.value = await response.json()
        metadata.value = {
          ...data.value.metadata,
          pollutants: ['ALL', ...data.value.metadata.pollutants]
        }
        stations.value = data.value.stations
        selectedPollutant.value = metadata.value.pollutants[0]
        timeRange.value = `${metadata.value.dateRange.start} to ${metadata.value.dateRange.end}`
      } catch (error) {
        console.error('Error loading data:', error)
      }
    }

    const drawTimeSeries = () => {
      if (!data.value || !timeSeriesChart.value) return

      const margin = { top: 20, right: 120, bottom: 30, left: 60 }
      const width = timeSeriesChart.value.clientWidth - margin.left - margin.right
      const height = 400 - margin.top - margin.bottom

      d3.select(timeSeriesChart.value).selectAll('*').remove()

      const svg = d3.select(timeSeriesChart.value)
        .append('svg')
        .attr('width', width + margin.left + margin.right)
        .attr('height', height + margin.top + margin.bottom)
        .append('g')
        .attr('transform', `translate(${margin.left},${margin.top})`)

      const pollutantColors = {
        'PM2.5': '#FF6B6B',
        'PM10': '#4ECDC4',
        'NO2': '#45B7D1',
        'SO2': '#96CEB4',
        'CO': '#FFEEAD',
        'O3': '#D4A5A5'
      }

      let filteredData
      if (selectedPollutant.value === 'ALL') {
        filteredData = data.value.timeSeriesData
      } else {
        filteredData = data.value.timeSeriesData
          .filter(d => d.Pollutant === selectedPollutant.value)
      }
      
      filteredData = filteredData.sort((a, b) => new Date(a.Date) - new Date(b.Date))

      const x = d3.scaleTime()
        .domain(d3.extent(filteredData, d => new Date(d.Date)))
        .range([0, width])

      const y = d3.scaleLinear()
        .domain([0, d3.max(filteredData, d => +d.Concentration)])
        .range([height, 0])
        .nice()

      svg.append('g')
        .attr('transform', `translate(0,${height})`)
        .call(d3.axisBottom(x).ticks(10))
        .selectAll('text')
        .style('text-anchor', 'middle')

      svg.append('g')
        .call(d3.axisLeft(y))

      svg.append('text')
        .attr('transform', 'rotate(-90)')
        .attr('y', 0 - margin.left)
        .attr('x', 0 - (height / 2))
        .attr('dy', '1em')
        .style('text-anchor', 'middle')
        .text('Concentration')

      if (selectedPollutant.value === 'ALL') {
        const pollutants = [...new Set(filteredData.map(d => d.Pollutant))]
        
        pollutants.forEach(pollutant => {
          const pollutantData = filteredData.filter(d => d.Pollutant === pollutant)
          
          const line = d3.line()
            .x(d => x(new Date(d.Date)))
            .y(d => y(+d.Concentration))
            .curve(d3.curveMonotoneX)

          const path = svg.append('path')
            .datum(pollutantData)
            .attr('fill', 'none')
            .attr('stroke', pollutantColors[pollutant])
            .attr('stroke-width', 2)
            .attr('d', line)

          const pathLength = path.node().getTotalLength()
          path.attr('stroke-dasharray', pathLength)
            .attr('stroke-dashoffset', pathLength)
            .transition()
            .duration(2000)
            .attr('stroke-dashoffset', 0)
        })

        const legend = svg.append('g')
          .attr('class', 'legend')
          .attr('transform', `translate(${width + 10}, 0)`)

        const legendItems = legend.selectAll('.legend-item')
          .data(pollutants)
          .enter()
          .append('g')
          .attr('class', 'legend-item')
          .attr('transform', (d, i) => `translate(0, ${i * 20})`)

        legendItems.append('line')
          .attr('x1', 0)
          .attr('x2', 20)
          .attr('y1', 10)
          .attr('y2', 10)
          .attr('stroke', d => pollutantColors[d])
          .attr('stroke-width', 2)

        legendItems.append('text')
          .attr('x', 25)
          .attr('y', 10)
          .attr('dy', '.35em')
          .style('font-size', '12px')
          .text(d => d)

      } else {
        const line = d3.line()
          .x(d => x(new Date(d.Date)))
          .y(d => y(+d.Concentration))
          .curve(d3.curveMonotoneX)

        const path = svg.append('path')
          .datum(filteredData)
          .attr('fill', 'none')
          .attr('stroke', pollutantColors[selectedPollutant.value])
          .attr('stroke-width', 2)
          .attr('d', line)

        const pathLength = path.node().getTotalLength()
        path.attr('stroke-dasharray', pathLength)
          .attr('stroke-dashoffset', pathLength)
          .transition()
          .duration(2000)
          .attr('stroke-dashoffset', 0)

        const dots = svg.selectAll('.dot')
          .data(filteredData)
          .enter()
          .append('circle')
          .attr('class', 'dot')
          .attr('cx', d => x(new Date(d.Date)))
          .attr('cy', d => y(+d.Concentration))
          .attr('r', 4)
          .attr('fill', pollutantColors[selectedPollutant.value])
          .style('opacity', 0)

        dots.transition()
          .delay((d, i) => i * 10)
          .style('opacity', 1)

        const tooltip = d3.select(timeSeriesChart.value)
          .append('div')
          .attr('class', 'tooltip')
          .style('opacity', 0)
          .style('position', 'absolute')
          .style('background-color', 'white')
          .style('border', '1px solid #ddd')
          .style('padding', '10px')
          .style('border-radius', '5px')
          .style('pointer-events', 'none')
          .style('z-index', 100)

        dots.on('mouseover', (event, d) => {
          const chartRect = timeSeriesChart.value.getBoundingClientRect()
          const mouseX = event.clientX - chartRect.left
          const mouseY = event.clientY - chartRect.top

          tooltip.transition()
            .duration(200)
            .style('opacity', .9)
          
          tooltip.html(`
            Date: ${d.Date}<br/>
            Concentration: ${d.Concentration} ${d.Units}<br/>
            AQI: ${d['Daily AQI Value']}
          `)
            .style('left', `${mouseX + 10}px`)
            .style('top', `${mouseY - 28}px`)


          d3.select(event.target)
            .transition()
            .duration(200)
            .attr('r', 6)
        })
        .on('mousemove', (event) => {
          const chartRect = timeSeriesChart.value.getBoundingClientRect()
          const mouseX = event.clientX - chartRect.left
          const mouseY = event.clientY - chartRect.top

          tooltip
            .style('left', `${mouseX + 10}px`)
            .style('top', `${mouseY - 28}px`)
        })
        .on('mouseout', (event) => {
          tooltip.transition()
            .duration(500)
            .style('opacity', 0)

          d3.select(event.target)
            .transition()
            .duration(200)
            .attr('r', 4)
        })
      }
    }

    const drawMap = async () => {
      if (!data.value || !mapChart.value) return

      const margin = { top: 20, right: 20, bottom: 20, left: 20 }
      const width = mapChart.value.clientWidth - margin.left - margin.right
      const height = 400 - margin.top - margin.bottom

      d3.select(mapChart.value).selectAll('*').remove()

      const svg = d3.select(mapChart.value)
        .append('svg')
        .attr('width', width + margin.left + margin.right)
        .attr('height', height + margin.top + margin.bottom)
        .append('g')
        .attr('transform', `translate(${margin.left},${margin.top})`)

      try {
        // Load map data
        const response = await fetch('/geo/los-angeles-county.geojson')
        const laCounty = await response.json()

        const projection = d3.geoMercator()
          .fitSize([width, height], laCounty)

        const path = d3.geoPath()
          .projection(projection)

        svg.append('g')
          .selectAll('path')
          .data(laCounty.features)
          .enter()
          .append('path')
          .attr('d', path)
          .attr('fill', '#f0f0f0')
          .attr('stroke', '#ccc')
          .attr('stroke-width', 0.5)

        const stationGroups = svg.append('g')
          .selectAll('.station')
          .data(stations.value)
          .enter()
          .append('g')
          .attr('class', 'station')
          .attr('transform', d => {
            const [x, y] = projection([+d['Site Longitude'], +d['Site Latitude']])
            return `translate(${x},${y})`
          })

        stationGroups.append('circle')
          .attr('r', 8)
          .attr('fill', d => d['Site ID'] === selectedStation.value ? '#f44336' : '#2196F3')
          .attr('opacity', 0)
          .transition()
          .duration(1000)
          .attr('opacity', 0.7)

        stationGroups
          .on('mouseover', function(event, d) {

            d3.select(this)
              .append('text')
              .attr('class', 'station-label')
              .attr('dy', -10)
              .text(d['Local Site Name'])
              .style('text-anchor', 'middle')
              .style('font-size', '12px')
              .style('opacity', 0)
              .transition()
              .duration(200)
              .style('opacity', 1)

            const tooltip = d3.select(mapChart.value)
              .append('div')
              .attr('class', 'map-tooltip')
              .style('position', 'absolute')
              .style('background-color', 'white')
              .style('padding', '10px')
              .style('border-radius', '5px')
              .style('box-shadow', '0 2px 5px rgba(0,0,0,0.2)')
              .style('pointer-events', 'none')
              .style('opacity', 0)

            tooltip.html(`
              <strong>${d['Local Site Name']}</strong><br/>
              Longitude: ${d['Site Longitude']}<br/>
              Latitude: ${d['Site Latitude']}<br/>
              Monitored Pollutants: ${d['Pollutant'].length} types
            `)
              .style('left', (event.pageX + 10) + 'px')
              .style('top', (event.pageY - 10) + 'px')
              .transition()
              .duration(200)
              .style('opacity', 1)

            d3.select(this).select('circle')
              .transition()
              .duration(200)
              .attr('r', 12)
          })
          .on('mouseout', function(event, d) {
            d3.select(this).select('.station-label').remove()

            d3.select(mapChart.value)
              .selectAll('.map-tooltip')
              .transition()
              .duration(200)
              .style('opacity', 0)
              .remove()

            if (d['Site ID'] !== selectedStation.value) {
              d3.select(this).select('circle')
                .transition()
                .duration(200)
                .attr('r', 8)
                .attr('fill', '#2196F3')
                .attr('opacity', 0.7)
            } else {
              d3.select(this).select('circle')
                .transition()
                .duration(200)
                .attr('r', 12)
                .attr('fill', '#f44336')
                .attr('opacity', 0.7)
            }
          })
          .on('click', (event, d) => {
            stationGroups.selectAll('circle')
              .transition()
              .duration(200)
              .attr('fill', '#2196F3')
              .attr('r', 8)
              .attr('opacity', 0.7)

            d3.select(event.currentTarget)
              .select('circle')
              .transition()
              .duration(200)
              .attr('fill', '#f44336')
              .attr('r', 12)
              .attr('opacity', 0.7)

            selectedStation.value = d['Site ID']
            handleStationChange()
          })

        const zoom = d3.zoom()
          .scaleExtent([1, 8])
          .on('zoom', (event) => {
            svg.selectAll('path')
              .attr('transform', event.transform)
            svg.selectAll('.station')
              .attr('transform', function(d) {
                const [x, y] = projection([+d['Site Longitude'], +d['Site Latitude']])
                return `translate(${event.transform.applyX(x)},${event.transform.applyY(y)})`
              })
          })

        svg.call(zoom)

      } catch (error) {
        console.error('Error loading map data:', error)
        svg.append('text')
          .attr('x', width / 2)
          .attr('y', height / 2)
          .attr('text-anchor', 'middle')
          .text('Error loading map data')
      }
    }

    const drawBarChart = () => {
      if (!data.value || !barChart.value) return

      const margin = { top: 20, right: 20, bottom: 40, left: 40 }
      const width = barChart.value.clientWidth - margin.left - margin.right
      const height = 400 - margin.top - margin.bottom

      d3.select(barChart.value).selectAll('*').remove()

      const svg = d3.select(barChart.value)
        .append('svg')
        .attr('width', width + margin.left + margin.right)
        .attr('height', height + margin.top + margin.bottom)
        .append('g')
        .attr('transform', `translate(${margin.left},${margin.top})`)

      let stationStats = data.value.stationStats
      if (selectedStation.value) {
        stationStats = stationStats.filter(stat => stat['Site ID'] === selectedStation.value)
      }

      const allPollutants = metadata.value.pollutants.filter(p => p !== 'ALL')
      
      let pollutantCounts = {}
      allPollutants.forEach(pollutant => {
        pollutantCounts[pollutant] = 0
      })

      stationStats.forEach(stat => {
        if (stat.Pollutant in pollutantCounts) {
          pollutantCounts[stat.Pollutant]++
        }
      })

      const totalStations = selectedStation.value ? 1 : stations.value.length

      const barData = allPollutants.map(pollutant => ({
        pollutant,
        count: pollutantCounts[pollutant],
        percentage: (pollutantCounts[pollutant] / totalStations) * 100
      }))

      // Create scales
      const x = d3.scaleBand()
        .range([0, width])
        .domain(allPollutants)
        .padding(0.3)

      const y = d3.scaleLinear()
        .range([height, 0])
        .domain([0, 100])

      svg.append('g')
        .attr('transform', `translate(0,${height})`)
        .call(d3.axisBottom(x))
        .selectAll('text')
        .style('text-anchor', 'end')
        .attr('dx', '-.8em')
        .attr('dy', '.15em')
        .attr('transform', 'rotate(-45)')

      svg.append('g')
        .call(d3.axisLeft(y).tickFormat(d => d + '%'))

      const bars = svg.selectAll('.bar')
        .data(barData)
        .enter()
        .append('rect')
        .attr('class', 'bar')
        .attr('x', d => x(d.pollutant))
        .attr('width', x.bandwidth())
        .attr('y', height)
        .attr('height', 0)
        .attr('fill', d => d.pollutant === selectedPollutant.value ? '#f44336' : '#2196F3')

      bars.transition()
        .duration(1000)
        .attr('y', d => y(d.percentage))
        .attr('height', d => height - y(d.percentage))

      svg.selectAll('.bar-label')
        .data(barData)
        .enter()
        .append('text')
        .attr('class', 'bar-label')
        .attr('x', d => x(d.pollutant) + x.bandwidth() / 2)
        .attr('y', d => y(d.percentage) - 5)
        .attr('text-anchor', 'middle')
        .text(d => `${d.percentage.toFixed(1)}%`)
        .style('opacity', 0)
        .transition()
        .duration(1000)
        .style('opacity', 1)
    }

    const drawPieChart = () => {
      if (!data.value || !pieChart.value) return

      const margin = { top: 20, right: 160, bottom: 20, left: 20 }
      const width = pieChart.value.clientWidth - margin.left - margin.right
      const height = 400 - margin.top - margin.bottom
      const radius = Math.min(width, height) / 2

      d3.select(pieChart.value).selectAll('*').remove()

      const svg = d3.select(pieChart.value)
        .append('svg')
        .attr('width', width + margin.left + margin.right)
        .attr('height', height + margin.top + margin.bottom)
        .append('g')
        .attr('transform', `translate(${width/2 + margin.left},${height/2 + margin.top})`)

      let filteredData = data.value.timeSeriesData
      if (selectedStation.value) {
        filteredData = filteredData.filter(d => d['Site ID'] === selectedStation.value)
      }

      const agiRanges = [
        { name: 'Good', range: '0-50', min: 0, max: 50, color: '#00e400' },
        { name: 'Moderate', range: '51-100', min: 51, max: 100, color: '#ffff00' },
        { name: 'Unhealthy for Sensitive Groups', range: '101-150', min: 101, max: 150, color: '#ff7e00' },
        { name: 'Unhealthy', range: '151-200', min: 151, max: 200, color: '#ff0000' },
        { name: 'Very Unhealthy', range: '201-300', min: 201, max: 300, color: '#99004c' },
        { name: 'Hazardous', range: '>300', min: 301, max: Infinity, color: '#7e0023' }
      ]

      const pieData = agiRanges.map(range => ({
        ...range,
        value: filteredData.filter(d => 
          d['Daily AQI Value'] >= range.min && 
          d['Daily AQI Value'] <= range.max
        ).length
      })).filter(d => d.value > 0)

      const total = d3.sum(pieData, d => d.value)

      const pie = d3.pie()
        .value(d => d.value)
        .sort(null)

      const arc = d3.arc()
        .innerRadius(radius * 0.4)
        .outerRadius(radius * 0.8)

      const arcs = svg.selectAll('.arc')
        .data(pie(pieData))
        .enter()
        .append('g')
        .attr('class', 'arc')

      arcs.append('path')
        .attr('d', arc)
        .attr('fill', d => d.data.color)
        .attr('stroke', 'white')
        .style('stroke-width', '2px')
        .style('opacity', 0)
        .transition()
        .duration(1000)
        .style('opacity', 1)
        .attrTween('d', function(d) {
          const interpolate = d3.interpolate({ startAngle: 0, endAngle: 0 }, d)
          return function(t) {
            return arc(interpolate(t))
          }
        })

      const legend = svg.append('g')
        .attr('transform', `translate(${radius + 30}, ${-radius})`)

      const legendItems = legend.selectAll('.legend-item')
        .data(pieData)
        .enter()
        .append('g')
        .attr('class', 'legend-item')
        .attr('transform', (d, i) => `translate(0, ${i * 60})`)

      legendItems.append('rect')
        .attr('width', 20)
        .attr('height', 20)
        .attr('fill', d => d.color)
        .style('opacity', 0)
        .transition()
        .duration(1000)
        .style('opacity', 1)

      const legendText = legendItems.append('text')
        .attr('x', 30)
        .attr('y', 15)
        .style('font-size', '12px')
        .style('opacity', 0)

      legendText.append('tspan')
        .text(d => `${d.name} (${d.range})`)
        .style('font-weight', 'bold')

      legendText.append('tspan')
        .attr('x', 30)
        .attr('dy', '1.2em')
        .text(d => `${d.value} times (${(d.value/total*100).toFixed(1)}%)`)

      legendText.transition()
        .delay(1000)
        .duration(500)
        .style('opacity', 1)

      const centerText = svg.append('text')
        .attr('text-anchor', 'middle')
        .style('opacity', 0)

      centerText.append('tspan')
        .text('AQI')
        .attr('x', 0)
        .attr('dy', '0em')
        .style('font-size', '24px')
        .style('font-weight', 'bold')

      centerText.append('tspan')
        .text('Distribution')
        .attr('x', 0)
        .attr('dy', '1.2em')
        .style('font-size', '18px')

      centerText.transition()
        .delay(1500)
        .duration(500)
        .style('opacity', 1)
    }

    const handleStationChange = () => {
      drawPieChart()
      drawBarChart()
    }

    watch(selectedPollutant, () => {
      drawTimeSeries()
      drawBarChart()
    })

    const handleResize = () => {
      drawTimeSeries()
      drawMap()
      drawBarChart()
      drawPieChart()
    }

    onMounted(async () => {
      await loadData()
      drawTimeSeries()
      drawMap()
      drawBarChart()
      drawPieChart()
      window.addEventListener('resize', handleResize)
    })

    return {
      timeSeriesChart,
      mapChart,
      barChart,
      pieChart,
      selectedPollutant,
      selectedStation,
      stations,
      metadata,
      timeRange,
      handleStationChange
    }
  }
}
</script>

<style>
.container {
 max-width: 1400px;
 margin: 0 auto;
 padding: 20px;
 font-family: Arial, sans-serif;
 background-color: #f5f5f5;
}

.header {
 text-align: center;
 margin-bottom: 30px;
}

.header h1 {
 color: #333;
 margin-bottom: 10px;
}

.row {
 margin-bottom: 30px;
}

.two-charts {
 display: grid;
 grid-template-columns: 1fr 1fr;
 gap: 20px;
}

.card {
 background: white;
 border-radius: 10px;
 box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
 padding: 20px;
 transition: all 0.3s ease;
}

.card:hover {
 transform: translateY(-2px);
 box-shadow: 0 4px 15px rgba(0, 0, 0, 0.15);
}

.card h2 {
 color: #333;
 margin-bottom: 20px;
 font-size: 1.2em;
}

.chart {
 width: 100%;
 height: 400px;
 position: relative;
}

.pollutant-selector,
.station-selector {
 margin-bottom: 15px;
 text-align: left;
}

select {
 padding: 8px 12px;
 border-radius: 5px;
 border: 1px solid #ddd;
 font-size: 14px;
 background-color: white;
 min-width: 200px;
 cursor: pointer;
 transition: all 0.3s ease;
}

select:hover {
 border-color: #2196F3;
}

select:focus {
 outline: none;
 border-color: #2196F3;
 box-shadow: 0 0 0 2px rgba(33, 150, 243, 0.2);
}

label {
 margin-right: 10px;
 color: #666;
}

.tooltip {
 font-size: 12px;
 pointer-events: none;
 z-index: 100;
 box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
 line-height: 1.5;
}

.dot:hover {
 cursor: pointer;
}

/* Add responsive layout */
@media (max-width: 1200px) {
 .container {
   padding: 15px;
 }
 
 .card {
   padding: 15px;
 }
}

@media (max-width: 768px) {
 .two-charts {
   grid-template-columns: 1fr;
 }
 
 .chart {
   height: 300px;
 }
 
 .card h2 {
   font-size: 1.1em;
 }
 
 select {
   width: 100%;
   max-width: none;
 }
}

@media (max-width: 480px) {
 .header h1 {
   font-size: 1.5em;
 }
 
 .chart {
   height: 250px;
 }
}

.map-tooltip {
  z-index: 100;
  font-size: 12px;
  line-height: 1.4;
}
</style>