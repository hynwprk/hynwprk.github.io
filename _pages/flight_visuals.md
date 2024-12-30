---
layout: page
title: "Flight Visuals"
permalink: /flight-visuals/
---

<!-- 
  Dark-themed “Flighty-like” page:
    1) Reads airport-locations.csv for lat/lon data.
    2) Reads FlightyExport-2024-12-30.csv for flight data.
    3) Correct distance (Haversine).
    4) 3D globe with routes, stats, chart, table.
-->

<style>
/* ===================== DARK THEME STYLING ===================== */

/* General page background and text color */
body {
  background-color: #1e1e1e;
  color: #eee;
  font-family: "Helvetica Neue", Helvetica, Arial, sans-serif;
  margin: 0;
  padding: 0;
}

/* Section containers */
.flight-section {
  margin: 2rem auto;
  padding: 1.5rem;
  max-width: 1100px;
  background-color: #2a2a2a;
  border: 1px solid #444;
  border-radius: 6px;
}

/* Headings */
.flight-section h2 {
  margin-top: 0;
  font-size: 1.4rem;
  color: #fff;
  border-bottom: 1px solid #555;
  padding-bottom: 0.5rem;
}

/* 3D globe container */
#cesiumContainer {
  width: 100%;
  height: 550px;
  border: 1px solid #555;
  border-radius: 4px;
  margin-top: 1rem;
}

/* Stats (cards) */
.stats-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
  margin-top: 1rem;
}

.stat-card {
  flex: 1 1 250px;
  background-color: #3a3a3a;
  border: 1px solid #555;
  border-radius: 6px;
  padding: 1rem;
  text-align: center;
}

.stat-card h3 {
  margin: 0 0 0.5rem 0;
  font-size: 1.2rem;
  color: #fff;
}

.stat-card p {
  margin: 0;
  font-size: 1rem;
  color: #ccc;
}

/* Chart */
#flightChart {
  max-width: 100%;
  margin: 1rem 0;
}

/* Table for flight listing */
#flightsTable {
  width: 100%;
  border-collapse: collapse;
  margin-top: 1rem;
  font-size: 0.9rem;
}

#flightsTable th,
#flightsTable td {
  border: 1px solid #555;
  padding: 0.5rem;
  text-align: left;
  color: #eee;
}

#flightsTable th {
  background-color: #444;
  color: #fff;
}
</style>

<!-- ========== 3D GLOBE SECTION AT THE TOP ========== -->
<div class="flight-section" id="flight-globe">
  <h2>Global Flight Routes</h2>
  <div id="cesiumContainer"></div>
</div>

<!-- ========== FLIGHT SUMMARY SECTION ========== -->
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
      <h3 id="times-around-world">0</h3>
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

<!-- ========== MONTHLY FLIGHT CHART ========== -->
<div class="flight-section" id="flight-chart-section">
  <h2>Monthly Flights Trend</h2>
  <canvas id="flightChart" width="600" height="300"></canvas>
</div>

<!-- ========== FLIGHT TABLE (OPTIONAL) ========== -->
<div class="flight-section" id="flight-table">
  <h2>All Flights</h2>
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
<script src="https://cdn.jsdelivr.net/npm/papaparse@5.4.1/papaparse.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<link
  rel="stylesheet"
  href="https://cdn.jsdelivr.net/npm/cesium@latest/Build/Cesium/Widgets/widgets.css"
/>
<script src="https://cdn.jsdelivr.net/npm/cesium@latest/Build/Cesium/Cesium.js"></script>

<script>
document.addEventListener('DOMContentLoaded', function() {
  // 1) We'll load airport-locations.csv to build airportDB
  const airportLocCSV = "{{ '/assets/data/airport-locations.csv' | relative_url }}";
  // 2) We'll also load FlightyExport-2024-12-30.csv
  const flightDataCSV = "{{ '/assets/data/FlightyExport-2024-12-30.csv' | relative_url }}";

  let airportDB = {}; // code -> [lat, lon]

  // Step A: Parse airport-locations.csv to build airportDB
  Papa.parse(airportLocCSV, {
    download: true,
    header: true,
    complete: function(results) {
      results.data.forEach(row => {
        // row.Orig is the IATA code, row.Airport1Latitude, row.Airport1Longitude
        let code = row.Orig; 
        let lat = parseFloat(row.Airport1Latitude);
        let lon = parseFloat(row.Airport1Longitude);
        if (!isNaN(lat) && !isNaN(lon) && code) {
          airportDB[code] = [lat, lon];
        }
      });
      // Now parse flight data
      parseFlightData();
    }
  });

  function parseFlightData() {
    Papa.parse(flightDataCSV, {
      download: true,
      header: true,
      complete: function(results) {
        const flights = results.data.filter(f => f.Date);
        buildVisualization(flights);
      }
    });
  }

  // Earth radius in miles
  const EARTH_RADIUS_MI = 3958.8;
  // Earth's circumference in miles
  const EARTH_CIRCUM_MI = 24901;
  let viewer;

  function buildVisualization(flights) {
    // 1) Initialize Cesium
    viewer = new Cesium.Viewer("cesiumContainer", {
      animation: false,
      timeline: false,
      baseLayerPicker: true,
      geocoder: false
    });
    // Hide Ion credit
    viewer.cesiumWidget.creditContainer.style.display = "none";

    let totalFlights = flights.length;
    let totalDistance = 0;
    let totalFlightHours = 0;
    let delayMinSum = 0;
    let uniqueAirports = new Set();
    let uniqueAirlines = new Set();
    let aircraftCount = {};
    let flightsByMonth = {};
    let usedAirports = new Set();

    // 2) Loop flights
    flights.forEach(f => {
      let fromCode = f.From;
      let toCode = f.To;
      // Airlines
      if (f.Airline) uniqueAirlines.add(f.Airline);

      // Distance
      if (airportDB[fromCode] && airportDB[toCode]) {
        let dist = computeDistanceMiles(airportDB[fromCode], airportDB[toCode]);
        totalDistance += dist;
      }

      // Mark used
      if (fromCode) usedAirports.add(fromCode);
      if (toCode) usedAirports.add(toCode);
      if (fromCode) uniqueAirports.add(fromCode);
      if (toCode) uniqueAirports.add(toCode);

      // Flight time
      let depAct = new Date(f["Take off (Actual)"] || f["Gate Departure (Actual)"]);
      let arrAct = new Date(f["Landing (Actual)"] || f["Gate Arrival (Actual)"]);
      if (!isNaN(depAct) && !isNaN(arrAct) && arrAct > depAct) {
        let diffHrs = (arrAct - depAct) / (1000 * 60 * 60);
        totalFlightHours += diffHrs;
      }

      // Delays
      let schDep = new Date(f["Gate Departure (Scheduled)"]);
      let actDep = new Date(f["Gate Departure (Actual)"]);
      if (!isNaN(schDep) && !isNaN(actDep) && actDep > schDep) {
        let diffMin = (actDep - schDep) / (1000 * 60);
        delayMinSum += diffMin;
      }

      // Aircraft 
      let acType = f["Aircraft Type Name"] || "Unknown";
      aircraftCount[acType] = (aircraftCount[acType] || 0) + 1;

      // Monthly grouping
      let d = new Date(f.Date);
      if (!isNaN(d)) {
        let ym = d.getFullYear() + "-" + String(d.getMonth() + 1).padStart(2, "0");
        flightsByMonth[ym] = (flightsByMonth[ym] || 0) + 1;
      }
    });

    // Summaries
    let totalDistanceRounded = Math.round(totalDistance);
    let aroundWorld = (totalDistance / EARTH_CIRCUM_MI).toFixed(2);
    let delayHours = (delayMinSum / 60).toFixed(1);

    // Convert flight hours to d/h/m
    let days = Math.floor(totalFlightHours / 24);
    let hours = Math.floor(totalFlightHours % 24);
    let leftover = (totalFlightHours - days * 24 - hours).toFixed(1);
    let leftoverMins = Math.round(leftover * 60);
    let timeStr = "";
    if (days > 0) timeStr += `${days}d `;
    if (hours > 0) timeStr += `${hours}h `;
    if (leftoverMins > 0) timeStr += `${leftoverMins}m`;
    if (!timeStr) timeStr = "0h";

    // Most flown aircraft
    let mostFlown = "None";
    let maxCount = 0;
    for (let ac in aircraftCount) {
      if (aircraftCount[ac] > maxCount) {
        mostFlown = ac;
        maxCount = aircraftCount[ac];
      }
    }

    // Fill DOM
    document.getElementById("total-flights").textContent = totalFlights;
    document.getElementById("total-distance").textContent = totalDistanceRounded;
    document.getElementById("times-around-world").textContent = aroundWorld;
    document.getElementById("total-flight-time").textContent = timeStr;
    document.getElementById("unique-airports").textContent = uniqueAirports.size;
    document.getElementById("unique-airlines").textContent = uniqueAirlines.size;
    document.getElementById("delay-hours").textContent = delayHours;
    document.getElementById("most-flown-aircraft").textContent = mostFlown;

    // Build monthly chart
    buildMonthlyChart(flightsByMonth);

    // Build flight table
    buildFlightTable(flights);

    // Add airport dots on globe
    usedAirports.forEach(code => {
      let coords = airportDB[code];
      if (!coords) return;
      let [lat, lon] = coords;
      viewer.entities.add({
        name: code,
        position: Cesium.Cartesian3.fromDegrees(lon, lat),
        point: {
          pixelSize: 6,
          color: Cesium.Color.YELLOW.withAlpha(0.8),
          outlineColor: Cesium.Color.BLACK,
          outlineWidth: 1
        },
        label: {
          text: code,
          font: "14px sans-serif",
          fillColor: Cesium.Color.WHITE,
          style: Cesium.LabelStyle.FILL_AND_OUTLINE,
          outlineWidth: 2,
          pixelOffset: new Cesium.Cartesian2(0, -20)
        }
      });
    });

    // Draw flight lines
    flights.forEach(f => {
      let fromCode = f.From;
      let toCode = f.To;
      if (!airportDB[fromCode] || !airportDB[toCode]) return;
      let [lat1, lon1] = airportDB[fromCode];
      let [lat2, lon2] = airportDB[toCode];
      viewer.entities.add({
        polyline: {
          positions: Cesium.Cartesian3.fromDegreesArray([lon1, lat1, lon2, lat2]),
          width: 2,
          material: Cesium.Color.CYAN.withAlpha(0.7)
        }
      });
    });

    // Fly to global view
    viewer.scene.camera.flyHome(2.0);
  }

  // ========== Distance (Haversine) in miles ========== //
  function computeDistanceMiles([lat1, lon1], [lat2, lon2]) {
    let toRad = angle => (angle * Math.PI) / 180;
    let dLat = toRad(lat2 - lat1);
    let dLon = toRad(lon2 - lon1);
    let rLat1 = toRad(lat1);
    let rLat2 = toRad(lat2);

    let a =
      Math.sin(dLat / 2) * Math.sin(dLat / 2) +
      Math.sin(dLon / 2) * Math.sin(dLon / 2) * Math.cos(rLat1) * Math.cos(rLat2);
    let c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    return EARTH_RADIUS_MI * c;
  }

  // ========== MONTHLY CHART ========== //
  function buildMonthlyChart(flightsByMonth) {
    let labels = Object.keys(flightsByMonth).sort();
    let data = labels.map(k => flightsByMonth[k]);

    let ctx = document.getElementById("flightChart").getContext("2d");
    new Chart(ctx, {
      type: "line",
      data: {
        labels,
        datasets: [
          {
            label: "Flights per Month",
            data,
            borderColor: "rgba(255, 99, 132, 1)",
            backgroundColor: "rgba(255, 99, 132, 0.2)",
            fill: true,
            tension: 0.1
          }
        ]
      },
      options: {
        responsive: true,
        scales: {
          x: { 
            title: { display: true, text: "Month (YYYY-MM)", color: "#eee" },
            ticks: { color: "#eee" }
          },
          y: { 
            title: { display: true, text: "Flights", color: "#eee" },
            beginAtZero: true,
            ticks: { color: "#eee" }
          }
        },
        plugins: {
          legend: {
            labels: { color: "#eee" }
          }
        }
      }
    });
  }

  // ========== FLIGHT TABLE ========== //
  function buildFlightTable(flights) {
    let tbody = document.querySelector("#flightsTable tbody");
    tbody.innerHTML = "";

    flights.forEach(f => {
      let tr = document.createElement("tr");
      let fromCode = f.From;
      let toCode = f.To;
      let dist = 0;
      if (airportDB[fromCode] && airportDB[toCode]) {
        dist = computeDistanceMiles(airportDB[fromCode], airportDB[toCode]);
      }

      // Departure delay
      let depSch = new Date(f["Gate Departure (Scheduled)"]);
      let depAct = new Date(f["Gate Departure (Actual)"]);
      let depMin = 0;
      if (!isNaN(depSch) && !isNaN(depAct) && depAct > depSch) {
        depMin = (depAct - depSch) / (1000 * 60);
      }

      // Arrival delay
      let arrSch = new Date(f["Gate Arrival (Scheduled)"]);
      let arrAct = new Date(f["Gate Arrival (Actual)"]);
      let arrMin = 0;
      if (!isNaN(arrSch) && !isNaN(arrAct) && arrAct > arrSch) {
        arrMin = (arrAct - arrSch) / (1000 * 60);
      }

      let rowData = [
        f.Date,
        f.Airline,
        f.Flight,
        fromCode,
        toCode,
        dist.toFixed(0),
        f["Aircraft Type Name"] || "",
        depMin.toFixed(1),
        arrMin.toFixed(1)
      ];

      rowData.forEach(val => {
        let td = document.createElement("td");
        td.textContent = val;
        tr.appendChild(td);
      });

      tbody.appendChild(tr);
    });
  }
});
</script>
