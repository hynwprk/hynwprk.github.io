---
layout: page
title: "Flight Visuals"
permalink: /flight-visuals/
---

<!-- 
  Dark-themed “Flighty-like” page:
    - Loads airport-locations.csv for lat/lon
    - Loads FlightyExport-2024-12-30.csv for flights
    - Correct flight time: 
       if "Take off (Actual)" & "Landing (Actual)" => use them
       else fallback to "Gate Departure (Actual)" & "Gate Arrival (Actual)"
    - No monthly chart
    - Removes dep/arr delay columns
    - Summaries + 3D globe + simpler table 
-->

<style>
/* ===================== DARK THEME STYLING ===================== */

/* General page background / text */
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

/* Table for flights */
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
  </div>
</div>

<!-- ========== FLIGHT TABLE (NO DELAY COLUMNS) ========== -->
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
      </tr>
    </thead>
    <tbody></tbody>
  </table>
</div>

<!-- ========== SCRIPTS (CDN) ========== -->
<script src="https://cdn.jsdelivr.net/npm/papaparse@5.4.1/papaparse.min.js"></script>
<link
  rel="stylesheet"
  href="https://cdn.jsdelivr.net/npm/cesium@latest/Build/Cesium/Widgets/widgets.css"
/>
<script src="https://cdn.jsdelivr.net/npm/cesium@latest/Build/Cesium/Cesium.js"></script>

<script>
document.addEventListener('DOMContentLoaded', function() {
  // CSV paths
  const airportLocCSV = "{{ '/assets/data/airport-locations.csv' | relative_url }}";
  const flightDataCSV = "{{ '/assets/data/FlightyExport-2024-12-30.csv' | relative_url }}";

  // Airport DB => code => [lat, lon]
  let airportDB = {};

  // ========== Step A: Parse airport-locations.csv ==========
  Papa.parse(airportLocCSV, {
    download: true,
    header: true,
    complete: function(results) {
      results.data.forEach(row => {
        // row.Orig => code, row.Airport1Latitude => lat, row.Airport1Longitude => lon
        let code = row.Orig;
        let lat = parseFloat(row.Airport1Latitude);
        let lon = parseFloat(row.Airport1Longitude);
        if (!isNaN(lat) && !isNaN(lon) && code) {
          airportDB[code] = [lat, lon];
        }
      });
      // Then parse flight data
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

  // Earth radius & circumference in miles
  const EARTH_RADIUS_MI = 3958.8;
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
    viewer.cesiumWidget.creditContainer.style.display = "none";

    let totalFlights = flights.length;
    let totalDistance = 0;
    let totalFlightHours = 0;
    let uniqueAirports = new Set();
    let uniqueAirlines = new Set();

    // We'll track used airports to place dots
    let usedAirports = new Set();

    // 2) Loop flights, compute distance & flight time
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

      // **Correct flight time**:
      // Prefer "Take off (Actual)" → "Landing (Actual)"
      // fallback "Gate Departure (Actual)" → "Gate Arrival (Actual)"
      let depActStr = f["Take off (Actual)"] || f["Gate Departure (Actual)"];
      let arrActStr = f["Landing (Actual)"] || f["Gate Arrival (Actual)"];
      let dep = new Date(depActStr);
      let arr = new Date(arrActStr);

      if (!isNaN(dep) && !isNaN(arr) && arr > dep) {
        let diffHrs = (arr - dep) / (1000 * 60 * 60);
        totalFlightHours += diffHrs;
      }
    });

    // Summaries
    let totalDistanceRounded = Math.round(totalDistance);
    let timesAroundWorld = (totalDistance / EARTH_CIRCUM_MI).toFixed(2);

    // Convert flight hours to days/hours/min
    let days = Math.floor(totalFlightHours / 24);
    let hours = Math.floor(totalFlightHours % 24);
    let leftover = (totalFlightHours - (days * 24) - hours).toFixed(2);
    let leftoverMins = Math.round(parseFloat(leftover) * 60);
    let timeStr = "";
    if (days > 0) timeStr += `${days}d `;
    if (hours > 0) timeStr += `${hours}h `;
    if (leftoverMins > 0) timeStr += `${leftoverMins}m`;
    if (!timeStr) timeStr = "0h";

    // Populate summary DOM
    document.getElementById("total-flights").textContent = totalFlights;
    document.getElementById("total-distance").textContent = totalDistanceRounded;
    document.getElementById("times-around-world").textContent = timesAroundWorld;
    document.getElementById("total-flight-time").textContent = timeStr;
    document.getElementById("unique-airports").textContent = uniqueAirports.size;
    document.getElementById("unique-airlines").textContent = uniqueAirlines.size;

    // Build table
    buildFlightTable(flights);

    // Plot airports
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

    // Plot flight routes
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
      Math.sin(dLat / 2) ** 2 +
      Math.sin(dLon / 2) ** 2 * Math.cos(rLat1) * Math.cos(rLat2);
    let c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    const EARTH_RADIUS_MI = 3958.8;
    return EARTH_RADIUS_MI * c;
  }

  // ========== Build Flight Table (No Dep/Arr Delay) ========== //
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

      let rowData = [
        f.Date,
        f.Airline,
        f.Flight,
        fromCode,
        toCode,
        dist.toFixed(0),
        f["Aircraft Type Name"] || ""
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
