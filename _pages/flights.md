---
layout: page
title: "Flights"
permalink: /flights/
---

<!-- 
  Dark-themed “Flights” page:
    - Summaries: total flights, total distance in “xxx K” style, times-around-world (+ suffix),
      #unique airports, #unique airlines, and a card for MOST VISITED AIRPORT (uppercase).
    - 3D globe labeled "Flights"
    - Table sorted newest → oldest
    - Day-of-year chart
    - Top-10 visited airports chart
-->

<style>
/* ===================== DARK THEME ===================== */
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

/* Chart containers */
.chart-container {
  margin-top: 1rem;
}

#topAirportsChart,
#dayOfYearChart {
  max-width: 100%;
  margin: 1rem 0;
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

<!-- ========== TOP 10 VISITED AIRPORTS ========== -->
<div class="flight-section" id="top-airports">
  <h2>Top 10 Visited Airports</h2>
  <p>Count how often each airport appears in From/To, then show the top 10.</p>
  <div class="chart-container">
    <canvas id="topAirportsChart"></canvas>
  </div>
</div>

<!-- ========== DAY-OF-YEAR DISTRIBUTION ========== -->
<div class="flight-section" id="day-of-year-dist">
  <h2>When Do I Fly in a Year?</h2>
  <p>Flights by day-of-year (1-365/366)</p>
  <div class="chart-container">
    <canvas id="dayOfYearChart"></canvas>
  </div>
</div>

<!-- ========== FLIGHT TABLE (Newest → Oldest) ========== -->
<div class="flight-section" id="flight-table">
  <h2>All Flights (New → Old)</h2>
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
  // Paths
  const airportLocCSV = "{{ '/assets/data/airport-locations.csv' | relative_url }}";
  const flightDataCSV = "{{ '/assets/data/FlightyExport-2024-12-30.csv' | relative_url }}";

  let airportDB = {};

  // 1) Parse airport-locations.csv
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

  function parseFlightData() {
    Papa.parse(flightDataCSV, {
      download: true,
      header: true,
      complete: function(results) {
        let flights = results.data.filter(f => f.Date);
        // Sort descending (newest → oldest)
        flights.sort((a, b) => parseDateUTC(b.Date) - parseDateUTC(a.Date));
        buildVisualization(flights);
      }
    });
  }

  // Earth
  const EARTH_RADIUS_MI = 3958.8;
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
    viewer.cesiumWidget.creditContainer.style.display = "none";

    // Summaries
    let totalFlights = flights.length;
    let totalDistance = 0;
    let uniqueAirports = new Set();
    let uniqueAirlines = new Set();
    let airportVisits = {}; // for top 10 & most visited airport
    let flightsByDayOfYear = {};
    for (let i = 1; i <= 366; i++) {
      flightsByDayOfYear[i] = 0;
    }

    flights.forEach(f => {
      let fromCode = f.From;
      let toCode = f.To;
      if (f.Airline) uniqueAirlines.add(f.Airline);

      // Distances
      if (airportDB[fromCode] && airportDB[toCode]) {
        let dist = computeDistanceMiles(airportDB[fromCode], airportDB[toCode]);
        totalDistance += dist;
      }

      // Track airports
      if (fromCode) {
        uniqueAirports.add(fromCode);
        airportVisits[fromCode] = (airportVisits[fromCode] || 0) + 1;
      }
      if (toCode) {
        uniqueAirports.add(toCode);
        airportVisits[toCode] = (airportVisits[toCode] || 0) + 1;
      }

      // Day-of-year chart
      let d = parseDateUTC(f.Date);
      if (d) {
        let dayOfYear = getDayOfYear(d);
        if (dayOfYear >= 1 && dayOfYear <= 366) {
          flightsByDayOfYear[dayOfYear]++;
        }
      }
    });

    // Summaries: totalDistance => "XXK" style
    // e.g. 143422 => 143 K
    let distK = Math.floor(totalDistance / 1000); 
    let distLabel = distK + "K";

    // Times around world => round down to 1 decimal => + suffix
    let rawTimes = (totalDistance / EARTH_CIRCUM_MI);
    let timesFloored = Math.floor(rawTimes * 10) / 10; // e.g. 2.3
    let timesLabel = timesFloored.toFixed(1) + "+";   // e.g. "2.3+"

    document.getElementById("total-flights").textContent = totalFlights;
    document.getElementById("total-distance").textContent = distLabel;
    document.getElementById("times-around-world").textContent = timesLabel;
    document.getElementById("unique-airports").textContent = uniqueAirports.size;
    document.getElementById("unique-airlines").textContent = uniqueAirlines.size;

    // Most visited airport (uppercase)
    let visitsArray = Object.entries(airportVisits); // [[code, count], ...]
    visitsArray.sort((a, b) => b[1] - a[1]);
    let mostVisited = visitsArray.length ? visitsArray[0][0].toUpperCase() : "---";
    document.getElementById("most-visited-airport").textContent = mostVisited;

    // Build table
    buildFlightTable(flights);

    // Plot on globe
    plotOnGlobe(flights, uniqueAirports);

    // Build charts
    buildTopAirportsChart(airportVisits);
    buildDayOfYearChart(flightsByDayOfYear);
  }

  // Parse date/time as UTC
  function parseDateUTC(dateStr) {
    if (!dateStr) return null;
    let str = dateStr.trim();
    // If no timezone indicated, forcibly append "Z" => treat as UTC
    if (!/[zZ]|([+\-]\d{2}:?\d{2})/.test(str)) {
      str += "Z";
    }
    let d = new Date(str);
    return isNaN(d) ? null : d;
  }

  // Day-of-year
  function getDayOfYear(dateObj) {
    let start = new Date(Date.UTC(dateObj.getUTCFullYear(), 0, 1));
    // difference in ms from Jan 1
    let diff = dateObj.getTime() - start.getTime();
    let day = Math.floor(diff / (1000 * 60 * 60 * 24)) + 1; 
    return day; // 1..365/366
  }

  // Distance (Haversine)
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
    return 3958.8 * c; // miles
  }

  // Plot flights on globe
  function plotOnGlobe(flights, airportSet) {
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

    viewer.scene.camera.flyHome(2.0);
  }

  // Build flight table
  function buildFlightTable(flights) {
    let tbody = document.querySelector("#flightsTable tbody");
    tbody.innerHTML = "";

    // flights is already sorted newest->oldest
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

  // Build top-10 airports chart
  function buildTopAirportsChart(airportVisits) {
    let visitsArray = Object.entries(airportVisits);
    visitsArray.sort((a, b) => b[1] - a[1]);
    let top10 = visitsArray.slice(0, 10);

    let labels = top10.map(item => item[0]);
    let data = top10.map(item => item[1]);

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
          x: {
            title: { display: true, text: "Airport", color: "#eee" },
            ticks: { color: "#eee" }
          },
          y: {
            title: { display: true, text: "Visits", color: "#eee" },
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

  // Build day-of-year chart
  function buildDayOfYearChart(flightsByDayOfYear) {
    let labels = [];
    let data = [];
    for (let day = 1; day <= 366; day++) {
      labels.push(day);
      data.push(flightsByDayOfYear[day]);
    }

    let ctx = document.getElementById("dayOfYearChart").getContext("2d");
    new Chart(ctx, {
      type: "bar",
      data: {
        labels,
        datasets: [
          {
            label: "Flights per Day of Year",
            data,
            backgroundColor: "rgba(255, 206, 86, 0.6)"
          }
        ]
      },
      options: {
        responsive: true,
        scales: {
          x: {
            title: { display: true, text: "Day of Year", color: "#eee" },
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
});
</script>
