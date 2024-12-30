---
layout: page
title: "Flight Visuals (3D Globe on Top)"
permalink: /flight-visuals/
---

<!-- 
  This page:
  1) Renders a 3D globe (Cesium) at the top
  2) Shows all used airports as dots
  3) Draws flight route polylines
  4) Displays summary stats, monthly chart, and optional flight table 
  in a pleasing style reminiscent of “Flighty.”
-->

<style>
/* ================ STYLING FOR A MODERN, CLEAN LOOK ================ */

/* Basic container styling for each major section */
.flight-section {
  margin: 2rem 0;
  padding: 1.5rem;
  background-color: #ffffff;
  border: 1px solid #ddd;
  border-radius: 6px;
  color: #333;
  font-family: "Helvetica Neue", Helvetica, Arial, sans-serif;
}

/* Section headings */
.flight-section h2 {
  margin-top: 0;
  font-size: 1.4rem;
  color: #2c3e50;
  border-bottom: 1px solid #ddd;
  padding-bottom: 0.5rem;
}

/* 3D globe container at top */
#cesiumContainer {
  width: 100%;
  height: 550px;
  border: 1px solid #ccc;
  border-radius: 4px;
  margin-top: 1rem;
}

/* Summary stats as cards */
.stats-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
  margin-top: 1rem;
}

.stat-card {
  flex: 1 1 250px;
  background-color: #f8f8f8;
  border: 1px solid #eee;
  border-radius: 6px;
  padding: 1rem;
  text-align: center;
}

.stat-card h3 {
  margin: 0 0 0.5rem 0;
  font-size: 1.2rem;
  color: #222;
}

.stat-card p {
  margin: 0;
  font-size: 1rem;
  color: #444;
}

/* Chart container */
#flightChart {
  max-width: 100%;
  margin: 1rem 0;
}

/* Optional table for flight listing */
#flightsTable {
  width: 100%;
  border-collapse: collapse;
  margin-top: 1rem;
  font-size: 0.9rem;
}

#flightsTable th,
#flightsTable td {
  border: 1px solid #ddd;
  padding: 0.5rem;
  text-align: left;
}

#flightsTable th {
  background-color: #f2f2f2;
}
</style>

<!-- ================== 3D GLOBE (CESIUM) SECTION AT THE TOP ================== -->
<div class="flight-section" id="flight-globe">
  <h2>Global Flight Routes</h2>
  <div id="cesiumContainer"></div>
</div>

<!-- ================== FLIGHT SUMMARY SECTION ================== -->
<div class="flight-section" id="flight-summary">
  <h2>Flight Summary</h2>
  <div class="stats-grid">
    <div class="stat-card">
      <h3 id="total-flights">0</h3>
      <p>Total Flights</p>
    </div>
    <div class="stat-card">
      <h3 id="total-distance">0</h3>
      <p>Total Distance (mi)</p>
    </div>
    <div class="stat-card">
      <h3 id="around-world">0</h3>
      <p>Times Around the World</p>
    </div>
    <div class="stat-card">
      <h3 id="total-flight-time">0</h3>
      <p>Total Flight Time</p>
    </div>
    <div class="stat-card">
      <h3 id="unique-airports">0</h3>
      <p>Airports Visited</p>
    </div>
    <div class="stat-card">
      <h3 id="unique-airlines">0</h3>
      <p>Airlines Flown</p>
    </div>
    <div class="stat-card">
      <h3 id="delay-hours">0</h3>
      <p>Hours Lost to Delays</p>
    </div>
    <div class="stat-card">
      <h3 id="most-flown-aircraft">—</h3>
      <p>Most Flown Aircraft</p>
    </div>
  </div>
</div>

<!-- ================== MONTHLY FLIGHT CHART ================== -->
<div class="flight-section" id="flight-charts">
  <h2>Monthly Flights Trend</h2>
  <canvas id="flightChart" width="600" height="300"></canvas>
</div>

<!-- ================== OPTIONAL FLIGHTS TABLE ================== -->
<div class="flight-section" id="flight-table">
  <h2>Flights (Optional Listing)</h2>
  <table id="flightsTable">
    <thead>
      <tr>
        <th>Date</th>
        <th>Airline</th>
        <th>Flight #</th>
        <th>From</th>
        <th>To</th>
        <th>Distance (mi)</th>
        <th>Aircraft</th>
        <th>Dep Delay (min)</th>
        <th>Arr Delay (min)</th>
      </tr>
    </thead>
    <tbody></tbody>
  </table>
</div>

<!-- ========== SCRIPTS (CDN) ========== -->
<!-- Papa Parse for CSV parsing -->
<script src="https://cdn.jsdelivr.net/npm/papaparse@5.4.1/papaparse.min.js"></script>
<!-- Chart.js for the monthly flight trend chart -->
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<!-- CesiumJS for the 3D globe -->
<link
  rel="stylesheet"
  href="https://cdn.jsdelivr.net/npm/cesium@latest/Build/Cesium/Widgets/widgets.css"
/>
<script src="https://cdn.jsdelivr.net/npm/cesium@latest/Build/Cesium/Cesium.js"></script>

<script>
document.addEventListener('DOMContentLoaded', function() {
  // 1) Path to CSV
  const csvPath = "{{ '/assets/data/FlightyExport-2024-12-30.csv' | relative_url }}";

  Papa.parse(csvPath, {
    download: true,
    header: true,
    complete: function(results) {
      const flights = results.data.filter(row => row.Date);
      processFlights(flights);
    }
  });

  // 2) AIRPORT COORDS - ensure all airports in your data are present
  const airportCoords = {
    // Example entries (you must add all your From/To codes):
    ICN: [37.4602, 126.4407],
    ATL: [33.6367, -84.4281],
    LGA: [40.7769, -73.8740],
    JFK: [40.6413, -73.7781],
    SFO: [37.6213, -122.3790],
    LAX: [33.9416, -118.4085],
    // ...
  };

  // Earth's radius in miles
  const EARTH_RADIUS_MI = 3958.8;
  // Earth's circumference in miles
  const EARTH_CIRCUM_MI = 24901;

  // We'll store some global references
  let viewer; // Cesium viewer

  function processFlights(flights) {
    // ========== Initialize the globe first ==========
    viewer = new Cesium.Viewer('cesiumContainer', {
      animation: false,
      timeline: false,
      baseLayerPicker: true,
      geocoder: false
    });
    // Optionally hide the Cesium ion credit
    viewer.cesiumWidget.creditContainer.style.display = 'none';

    // ========== Summaries / Stats Variables ==========
    let totalFlights = flights.length;
    let totalDistanceMi = 0;
    let totalFlightHours = 0;
    let departureDelayMinSum = 0;
    let mostFlownAircraftCount = {};
    let uniqueAirports = new Set();
    let uniqueAirlines = new Set();
    let flightsByMonth = {}; // for chart

    // We also want to track which airports we actually use, so we can place a point on the globe
    let usedAirports = new Set();

    flights.forEach(f => {
      // Unique Airlines
      if (f.Airline) uniqueAirlines.add(f.Airline);

      // Distances
      let fromCode = f.From;
      let toCode = f.To;
      if (airportCoords[fromCode] && airportCoords[toCode]) {
        let dist = computeDistanceMiles(airportCoords[fromCode], airportCoords[toCode]);
        totalDistanceMi += dist;
      }

      // Track airports for the globe
      if (fromCode) usedAirports.add(fromCode);
      if (toCode) usedAirports.add(toCode);

      // Unique airports set
      if (fromCode) uniqueAirports.add(fromCode);
      if (toCode) uniqueAirports.add(toCode);

      // Flight hours
      let depActual = new Date(f['Take off (Actual)'] || f['Gate Departure (Actual)']);
      let arrActual = new Date(f['Landing (Actual)'] || f['Gate Arrival (Actual)']);
      if (!isNaN(depActual) && !isNaN(arrActual) && arrActual > depActual) {
        let diffMs = arrActual - depActual;
        let hours = diffMs / (1000 * 60 * 60);
        totalFlightHours += hours;
      }

      // Departure delay
      let depSch = new Date(f['Gate Departure (Scheduled)']);
      let depAct = new Date(f['Gate Departure (Actual)']);
      if (!isNaN(depSch) && !isNaN(depAct) && depAct > depSch) {
        let diffMin = (depAct - depSch) / (1000 * 60);
        departureDelayMinSum += diffMin;
      }

      // Track aircraft usage
      let acType = f['Aircraft Type Name'] || 'Unknown';
      mostFlownAircraftCount[acType] = (mostFlownAircraftCount[acType] || 0) + 1;

      // Monthly grouping
      let d = new Date(f.Date);
      if (!isNaN(d)) {
        let ym = d.getFullYear() + '-' + String(d.getMonth() + 1).padStart(2, '0');
        flightsByMonth[ym] = (flightsByMonth[ym] || 0) + 1;
      }
    });

    // ========== Summaries ==========
    let totalDistanceRounded = Math.round(totalDistanceMi);
    let timesAroundWorld = (totalDistanceMi / EARTH_CIRCUM_MI).toFixed(2);
    let delayHours = (departureDelayMinSum / 60).toFixed(1);

    // Flight time in d/h/m
    let days = Math.floor(totalFlightHours / 24);
    let hours = Math.floor(totalFlightHours % 24);
    let leftover = (totalFlightHours - days * 24 - hours).toFixed(1);
    let leftoverMins = Math.round(leftover * 60);
    let timeStr = '';
    if (days > 0) timeStr += `${days}d `;
    if (hours > 0) timeStr += `${hours}h `;
    if (leftoverMins > 0) timeStr += `${leftoverMins}m`;
    if (!timeStr) timeStr = '0h';

    // Most flown aircraft
    let mostFlown = 'None';
    let maxCount = 0;
    for (let ac in mostFlownAircraftCount) {
      if (mostFlownAircraftCount[ac] > maxCount) {
        mostFlown = ac;
        maxCount = mostFlownAircraftCount[ac];
      }
    }

    // Populate summary cards
    document.getElementById('total-flights').textContent = totalFlights;
    document.getElementById('total-distance').textContent = totalDistanceRounded;
    document.getElementById('around-world').textContent = timesAroundWorld;
    document.getElementById('total-flight-time').textContent = timeStr;
    document.getElementById('unique-airports').textContent = uniqueAirports.size;
    document.getElementById('unique-airlines').textContent = uniqueAirlines.size;
    document.getElementById('delay-hours').textContent = delayHours;
    document.getElementById('most-flown-aircraft').textContent = mostFlown;

    // ========== Build monthly chart ==========
    buildMonthlyChart(flightsByMonth);

    // ========== Build optional table ==========
    buildFlightTable(flights);

    // ========== Add airports as dots on the globe ==========
    usedAirports.forEach(code => {
      if (!airportCoords[code]) return;
      let [lat, lon] = airportCoords[code];
      viewer.entities.add({
        name: code,
        position: Cesium.Cartesian3.fromDegrees(lon, lat),
        point: {
          pixelSize: 6,
          color: Cesium.Color.RED.withAlpha(0.8),
          outlineColor: Cesium.Color.WHITE,
          outlineWidth: 1
        },
        label: {
          text: code,
          font: '14px sans-serif',
          fillColor: Cesium.Color.WHITE,
          style: Cesium.LabelStyle.FILL_AND_OUTLINE,
          outlineWidth: 2,
          pixelOffset: new Cesium.Cartesian2(0, -18) // offset label above dot
        }
      });
    });

    // ========== Draw flight routes as polylines ==========
    flights.forEach(f => {
      let from = f.From, to = f.To;
      if (!airportCoords[from] || !airportCoords[to]) return;
      let [lat1, lon1] = airportCoords[from];
      let [lat2, lon2] = airportCoords[to];
      viewer.entities.add({
        polyline: {
          positions: Cesium.Cartesian3.fromDegreesArray([lon1, lat1, lon2, lat2]),
          width: 2,
          material: Cesium.Color.fromCssColorString('#007aff').withAlpha(0.7)
        }
      });
    });

    // Fly the camera to a global view
    viewer.scene.camera.flyHome(2.0);
  }

  // ========== DISTANCE (Haversine) ==========
  function computeDistanceMiles([lat1, lon1], [lat2, lon2]) {
    let toRad = angle => (angle * Math.PI) / 180;
    let dLat = toRad(lat2 - lat1);
    let dLon = toRad(lon2 - lon1);
    let rLat1 = toRad(lat1);
    let rLat2 = toRad(lat2);

    let a =
      Math.sin(dLat / 2) * Math.sin(dLat / 2) +
      Math.sin(dLon / 2) *
        Math.sin(dLon / 2) *
        Math.cos(rLat1) *
        Math.cos(rLat2);
    let c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    return EARTH_RADIUS_MI * c;
  }

  // ========== MONTHLY CHART (Chart.js) ==========
  function buildMonthlyChart(flightsByMonth) {
    let labels = Object.keys(flightsByMonth).sort();
    let data = labels.map(k => flightsByMonth[k]);

    let ctx = document.getElementById('flightChart').getContext('2d');
    new Chart(ctx, {
      type: 'line',
      data: {
        labels,
        datasets: [
          {
            label: 'Flights per Month',
            data,
            borderColor: 'rgba(54, 162, 235, 1)',
            backgroundColor: 'rgba(54, 162, 235, 0.2)',
            fill: true,
            tension: 0.1
          }
        ]
      },
      options: {
        responsive: true,
        scales: {
          x: { title: { display: true, text: 'Month (YYYY-MM)' } },
          y: { title: { display: true, text: 'Flights' }, beginAtZero: true }
        }
      }
    });
  }

  // ========== OPTIONAL FLIGHTS TABLE ==========
  function buildFlightTable(flights) {
    let tbody = document.querySelector('#flightsTable tbody');
    tbody.innerHTML = '';

    flights.forEach(f => {
      let tr = document.createElement('tr');

      let distanceMi = 0;
      if (airportCoords[f.From] && airportCoords[f.To]) {
        distanceMi = computeDistanceMiles(airportCoords[f.From], airportCoords[f.To]);
      }
      let depSch = new Date(f['Gate Departure (Scheduled)']);
      let depAct = new Date(f['Gate Departure (Actual)']);
      let arrSch = new Date(f['Gate Arrival (Scheduled)']);
      let arrAct = new Date(f['Gate Arrival (Actual)']);
      let depDelay = 0;
      let arrDelay = 0;

      if (!isNaN(depSch) && !isNaN(depAct) && depAct > depSch) {
        depDelay = (depAct - depSch) / (1000 * 60);
      }
      if (!isNaN(arrSch) && !isNaN(arrAct) && arrAct > arrSch) {
        arrDelay = (arrAct - arrSch) / (1000 * 60);
      }

      let rowData = [
        f.Date,
        f.Airline,
        f.Flight,
        f.From,
        f.To,
        distanceMi.toFixed(0),
        f['Aircraft Type Name'] || '',
        depDelay.toFixed(1),
        arrDelay.toFixed(1)
      ];

      rowData.forEach(val => {
        let td = document.createElement('td');
        td.textContent = val || '';
        tr.appendChild(td);
      });
      tbody.appendChild(tr);
    });
  }
});
</script>
