---
layout: page
title: "flights"
permalink: /flights/
---

<!-- 
If NOT using Jekyll's site.static_files, replace latestFlighty below
with a hardcoded path like '/assets/data/FlightyExport-2024-12-30.csv'
-->
{% assign flightyFiles = site.static_files | where_exp: "f","f.path contains 'FlightyExport-'" %}
{% assign sortedFlightyFiles = flightyFiles | sort: 'path' %}
{% assign latestFlighty = sortedFlightyFiles | last %}

<style>
/* ===================== DARK THEME ===================== */
body {
  background-color: #1e1e1e;
  color: #eee;
  font-family: "Helvetica Neue", Helvetica, Arial, sans-serif;
  margin: 0;
  padding: 0;
}

.flight-section {
  margin: 2rem auto;
  padding: 1.5rem;
  max-width: 1100px;
  background-color: #2a2a2a;
  border: 1px solid #444;
  border-radius: 6px;
}

.flight-section h2 {
  margin-top: 0;
  font-size: 1.4rem;
  color: #fff;
  border-bottom: 1px solid #555;
  padding-bottom: 0.5rem;
}

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

/* Charts */
.chart-container {
  margin-top: 1rem;
}

#topAirportsChart,
#topRoutesChart,
#weekdayChart {
  max-width: 100%;
  margin: 1rem 0;
}

/* Table */
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

<!-- ========== FLIGHTS (3D GLOBE) ========== -->
<div class="flight-section" id="flight-globe">
  <h2>Flights</h2>
  <div id="cesiumContainer"></div>
</div>

<!-- ========== SUMMARY SECTION ========== -->
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
      <h3 id="most-visited-airport">---</h3>
      <p>Most Visited Airport</p>
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

<!-- ========== TOP VISITED AIRPORTS (Top 8) ========== -->
<div class="flight-section" id="top-airports">
  <h2>Top 8 Visited Airports</h2>
  <div class="chart-container">
    <canvas id="topAirportsChart"></canvas>
  </div>
</div>

<!-- ========== TOP ROUTES (DIRECTIONAL, Top 8) ========== -->
<div class="flight-section" id="top-routes">
  <h2>Top Routes (Direction-Specific)</h2>
  <div class="chart-container">
    <canvas id="topRoutesChart"></canvas>
  </div>
</div>

<!-- ========== WEEKDAY CHART (SUN-SAT) ========== -->
<div class="flight-section" id="weekday-dist">
  <h2>Flights by Day of Week</h2>
  <div class="chart-container">
    <canvas id="weekdayChart"></canvas>
  </div>
</div>

<!-- ========== FLIGHT TABLE (NEW → OLD) ========== -->
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
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<link
  rel="stylesheet"
  href="https://cdn.jsdelivr.net/npm/cesium@latest/Build/Cesium/Widgets/widgets.css"
/>
<script src="https://cdn.jsdelivr.net/npm/cesium@latest/Build/Cesium/Cesium.js"></script>

<script>
document.addEventListener("DOMContentLoaded", function() {
  /*******************************************************
    1) Load airport-locations
  *******************************************************/
  const airportLocCSV = "{{ '/assets/data/airport-locations.csv' | relative_url }}";

  /*******************************************************
    2) The highest-dated Flighty CSV (or set your own path)
  *******************************************************/
  const flightDataCSV = "{{ latestFlighty.path | relative_url }}";

  let airportDB = {};

  // Parse airport CSV first
  Papa.parse(airportLocCSV, {
    download: true,
    header: true,
    complete: function(results) {
      results.data.forEach(row => {
        let code = row.Orig;
        let lat = parseFloat(row.Airport1Latitude);
        let lon = parseFloat(row.Airport1Longitude);
        if (!isNaN(lat) && !isNaN(lon) && code) {
          airportDB[code] = [lat, lon];
        }
      });
      parseFlightData();
    }
  });

  // Then parse the flight CSV
  function parseFlightData() {
    Papa.parse(flightDataCSV, {
      download: true,
      header: true,
      complete: function(results) {
        // Filter out rows missing Date
        let flights = results.data.filter(f => f.Date);

        // Sort ascending, then reverse => newest first
        flights.sort((a, b) => parseDateUTC(a.Date) - parseDateUTC(b.Date));
        flights.reverse();

        buildVisualization(flights);
      }
    });
  }

  /*******************************************************
    Earth circumference for "times around the world"
  *******************************************************/
  const EARTH_CIRCUM_MI = 24901;
  let viewer;

  function buildVisualization(flights) {
    // Initialize Cesium
    viewer = new Cesium.Viewer("cesiumContainer", {
      animation: false,
      timeline: false,
      baseLayerPicker: true,
      geocoder: false
    });
    // Hide default Cesium credits
    viewer.cesiumWidget.creditContainer.style.display = "none";

    let totalFlights = flights.length;
    let totalDistance = 0;
    let uniqueAirports = new Set();
    let uniqueAirlines = new Set();

    let airportVisits = {};
    let routeVisits = {};
    let flightsByWeekday = [0, 0, 0, 0, 0, 0, 0]; // index 0 => Sunday

    /****************************************************
      Tally all data
    ****************************************************/
    flights.forEach(f => {
      let fromCode = (f.From || "").trim();
      let toCode = (f.To || "").trim();

      // Distance
      if (airportDB[fromCode] && airportDB[toCode]) {
        let dist = computeDistanceMiles(airportDB[fromCode], airportDB[toCode]);
        totalDistance += dist;
      }

      // Airport visits
      if (fromCode) {
        uniqueAirports.add(fromCode);
        airportVisits[fromCode] = (airportVisits[fromCode] || 0) + 1;
      }
      if (toCode) {
        uniqueAirports.add(toCode);
        airportVisits[toCode] = (airportVisits[toCode] || 0) + 1;
      }

      // Directional routes (no unification)
      // e.g. from=ATL, to=ICN => routeKey="ATL-ICN"
      // if reversed, routeKey="ICN-ATL"
      if (fromCode && toCode && fromCode !== toCode) {
        let routeKey = fromCode.toUpperCase() + "-" + toCode.toUpperCase();
        routeVisits[routeKey] = (routeVisits[routeKey] || 0) + 1;
      }

      // Weekday
      let dt = parseDateUTC(f.Date);
      if (dt) {
        flightsByWeekday[dt.getUTCDay()]++;
      }

      // Airline
      if (f.Airline) {
        uniqueAirlines.add(f.Airline.trim());
      }
    });

    /****************************************************
      Summaries
    ****************************************************/
    document.getElementById("total-flights").textContent = totalFlights;
    document.getElementById("unique-airports").textContent = uniqueAirports.size;
    document.getElementById("unique-airlines").textContent = uniqueAirlines.size;

    // Distance
    let distK = Math.floor(totalDistance / 1000) + "k";
    document.getElementById("total-distance").textContent = distK;

    // Times around the world
    let rawTimes = totalDistance / EARTH_CIRCUM_MI;
    let timesFloored = Math.floor(rawTimes * 10) / 10;
    document.getElementById("times-around-world").textContent = timesFloored.toFixed(1);

    // Most visited airport
    let visitsArray = Object.entries(airportVisits).sort((a, b) => b[1] - a[1]);
    let mostVisited = visitsArray.length ? visitsArray[0][0].toUpperCase() : "---";
    document.getElementById("most-visited-airport").textContent = mostVisited;

    /****************************************************
      Build Table, Plot on Globe, Build Charts
    ****************************************************/
    buildFlightTable(flights);
    plotOnGlobe(flights, uniqueAirports);
    buildTopAirportsChart(airportVisits);
    buildTopRoutesChart(routeVisits);
    buildWeekdayChart(flightsByWeekday);
  }

  /*******************************************************
    Date Parsing (UTC)
  *******************************************************/
  function parseDateUTC(dateStr) {
    if (!dateStr) return null;
    let str = dateStr.trim();
    // If just YYYY-MM-DD, append T00:00:00Z
    if (/^\d{4}-\d{2}-\d{2}$/.test(str)) {
      str += "T00:00:00Z";
    } else if (!/[zZ]|([+\-]\d{2}:?\d{2})/.test(str)) {
      // If no timezone, assume UTC
      str += "Z";
    }
    let d = new Date(str);
    return isNaN(d) ? null : d;
  }

  /*******************************************************
    Haversine distance in miles
  *******************************************************/
  function computeDistanceMiles([lat1, lon1], [lat2, lon2]) {
    let toRad = a => (a * Math.PI) / 180;
    let dLat = toRad(lat2 - lat1);
    let dLon = toRad(lon2 - lon1);
    let rLat1 = toRad(lat1);
    let rLat2 = toRad(lat2);

    let a =
      Math.sin(dLat / 2) ** 2 +
      Math.sin(dLon / 2) ** 2 * Math.cos(rLat1) * Math.cos(rLat2);
    let c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    return 3958.8 * c;
  }

  /*******************************************************
    Plot on Cesium globe
  *******************************************************/
  function plotOnGlobe(flights, airportSet) {
    // Plot airports
    airportSet.forEach(code => {
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

    // Plot routes
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

    // Zoom out to show entire globe
    viewer.scene.camera.flyHome(2.0);
  }

  /*******************************************************
    Flight Table (reverse chronological is already done)
  *******************************************************/
  function buildFlightTable(flights) {
    let tbody = document.querySelector("#flightsTable tbody");
    tbody.innerHTML = "";

    flights.forEach(f => {
      let tr = document.createElement("tr");
      let dist = 0;
      if (airportDB[f.From] && airportDB[f.To]) {
        dist = computeDistanceMiles(airportDB[f.From], airportDB[f.To]);
      }
      let rowData = [
        f.Date,
        f.Airline,
        f.Flight,
        f.From,
        f.To,
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

  /*******************************************************
    CHARTS: Force y-axis to start at 0 with type=linear, min=0
  *******************************************************/
  // === Top 8 Airports (no unification needed) ===
  function buildTopAirportsChart(airportVisits) {
    let visitsArray = Object.entries(airportVisits).sort((a, b) => b[1] - a[1]);
    let top8 = visitsArray.slice(0, 8);

    let labels = top8.map(item => item[0]);
    let data = top8.map(item => item[1]);

    let ctx = document.getElementById("topAirportsChart").getContext("2d");
    new Chart(ctx, {
      type: "bar",
      data: {
        labels,
        datasets: [
          {
            label: "Visits",
            data,
            backgroundColor: "rgba(54, 162, 235, 0.6)"
          }
        ]
      },
      options: {
        responsive: true,
        scales: {
          y: {
            type: "linear",
            min: 0,
            beginAtZero: true,
            ticks: { color: "#eee" },
            title: { display: false }
          },
          x: {
            ticks: { color: "#eee" },
            title: { display: false }
          }
        },
        plugins: {
          legend: { display: false }
        }
      }
    });
  }

  // === Top 8 Routes (direction-specific) ===
  function buildTopRoutesChart(routeVisits) {
    // e.g. "ATL-ICN" vs "ICN-ATL" are counted separately
    let routesArray = Object.entries(routeVisits).sort((a, b) => b[1] - a[1]);
    let top8 = routesArray.slice(0, 8);

    let labels = top8.map(r => r[0]); 
    let data = top8.map(r => r[1]);

    let ctx = document.getElementById("topRoutesChart").getContext("2d");
    new Chart(ctx, {
      type: "bar",
      data: {
        labels,
        datasets: [
          {
            label: "Count",
            data,
            backgroundColor: "rgba(153, 102, 255, 0.6)"
          }
        ]
      },
      options: {
        responsive: true,
        scales: {
          y: {
            type: "linear",
            min: 0,
            beginAtZero: true,
            ticks: { color: "#eee" },
            title: { display: false }
          },
          x: {
            ticks: { color: "#eee" },
            title: { display: false }
          }
        },
        plugins: {
          legend: { display: false }
        }
      }
    });
  }

  // === Weekday Chart (Sun–Sat) ===
  function buildWeekdayChart(flightsByWeekday) {
    let labels = ["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"];
    let data = flightsByWeekday;

    let ctx = document.getElementById("weekdayChart").getContext("2d");
    new Chart(ctx, {
      type: "bar",
      data: {
        labels,
        datasets: [
          {
            label: "Flights",
            data,
            backgroundColor: "rgba(255, 159, 64, 0.6)"
          }
        ]
      },
      options: {
        responsive: true,
        scales: {
          y: {
            type: "linear",
            min: 0,
            beginAtZero: true,
            ticks: { color: "#eee" },
            title: { display: false }
          },
          x: {
            ticks: { color: "#eee" },
            title: { display: false }
          }
        },
        plugins: {
          legend: { display: false }
        }
      }
    });
  }
});
</script>
