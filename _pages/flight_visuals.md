---
layout: page
title: "My Flight Visualizations"
permalink: /flight-visuals/
---

<!-- 
  Flighty-like visualization 
  1) CSV flight data
  2) Summary stats
  3) Monthly chart
  4) Full table (with all columns)
  5) 3D globe (CesiumJS)
-->

<style>
  /* ========== FLIGHTY-LIKE, CLEAN STYLING ========== */
  #flight-summary,
  #flight-charts,
  #flight-table,
  #flight-globe {
    margin: 2rem 0;
    padding: 1.5rem;
    border: 1px solid #eee;
    border-radius: 6px;
    background-color: #fafafa;
  }

  #flight-summary h2,
  #flight-charts h2,
  #flight-table h2,
  #flight-globe h2 {
    margin-top: 0;
    font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif;
    font-size: 1.4rem;
    color: #2c3e50;
    border-bottom: 1px solid #ddd;
    padding-bottom: 0.5rem;
  }

  #flight-summary p {
    margin: 0.3rem 0;
  }

  #flightsTable {
    width: 100%;
    border-collapse: collapse;
    font-size: 0.9rem;
  }

  #flightsTable th,
  #flightsTable td {
    border-bottom: 1px solid #ddd;
    padding: 0.5rem;
    text-align: left;
  }

  #flightsByMonth {
    max-width: 100%;
  }

  /* The container that will hold the Cesium globe */
  #cesiumContainer {
    width: 100%;
    height: 600px;
    border: 1px solid #ccc;
    border-radius: 4px;
  }
</style>

<!-- ========== FLIGHT SUMMARY SECTION ========== -->
<div id="flight-summary">
  <h2>Flight Summary</h2>
  <p><strong>Total Flights:</strong> <span id="total-flights"></span></p>
  <p><strong>Total Flight Hours:</strong> <span id="total-flight-hours"></span></p>
  <p><strong>Average Departure Delay (mins):</strong> <span id="avg-dep-delay"></span></p>
  <p><strong>Average Arrival Delay (mins):</strong> <span id="avg-arr-delay"></span></p>
  <p><strong>Cancellations:</strong> <span id="total-cancellations"></span></p>
  <p><strong>Diversions:</strong> <span id="total-diversions"></span></p>
</div>

<!-- ========== MONTHLY FLIGHTS CHART ========== -->
<div id="flight-charts">
  <h2>Flights by Month</h2>
  <canvas id="flightsByMonth" width="600" height="300"></canvas>
</div>

<!-- ========== FLIGHTS TABLE ========== -->
<div id="flight-table">
  <h2>All Flights</h2>
  <table id="flightsTable">
    <thead>
      <tr>
        <!-- Let's list all columns from your CSV in the order you want them: -->
        <th>Date</th>
        <th>Airline</th>
        <th>Flight</th>
        <th>From</th>
        <th>To</th>
        <th>Dep Terminal</th>
        <th>Dep Gate</th>
        <th>Arr Terminal</th>
        <th>Arr Gate</th>
        <th>Canceled</th>
        <th>Diverted To</th>
        <th>Gate Departure (Scheduled)</th>
        <th>Gate Departure (Actual)</th>
        <th>Take off (Scheduled)</th>
        <th>Take off (Actual)</th>
        <th>Landing (Scheduled)</th>
        <th>Landing (Actual)</th>
        <th>Gate Arrival (Scheduled)</th>
        <th>Gate Arrival (Actual)</th>
        <th>Aircraft Type Name</th>
        <th>Tail Number</th>
        <th>PNR</th>
        <th>Seat</th>
        <th>Seat Type</th>
        <th>Cabin Class</th>
        <th>Flight Reason</th>
        <th>Notes</th>
        <th>Flight Flighty ID</th>
        <th>Airline Flighty ID</th>
        <th>Departure Airport Flighty ID</th>
        <th>Arrival Airport Flighty ID</th>
        <th>Diverted To Airport Flighty ID</th>
        <th>Aircraft Type Flighty ID</th>
      </tr>
    </thead>
    <tbody></tbody>
  </table>
</div>

<!-- ========== 3D GLOBE SECTION (CESIUM) ========== -->
<div id="flight-globe">
  <h2>3D Globe</h2>
  <div id="cesiumContainer"></div>
</div>

<!-- ========== SCRIPTS (CDN) ========== -->
<!-- Papa Parse to read CSV -->
<script src="https://cdn.jsdelivr.net/npm/papaparse@5.4.1/papaparse.min.js"></script>
<!-- Chart.js for monthly chart -->
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<!-- CesiumJS for 3D globe -->
<link
  rel="stylesheet"
  href="https://cdn.jsdelivr.net/npm/cesium@latest/Build/Cesium/Widgets/widgets.css"
/>
<script src="https://cdn.jsdelivr.net/npm/cesium@latest/Build/Cesium/Cesium.js"></script>

<script>
  document.addEventListener('DOMContentLoaded', function () {
    // 1) Path to your CSV file
    const csvPath = "{{ '/assets/data/FlightyExport-2024-12-30.csv' | relative_url }}";

    Papa.parse(csvPath, {
      download: true,
      header: true,
      complete: function (results) {
        const flights = results.data.filter((row) => row.Date); 
        buildSummary(flights);
        buildChart(flights);
        buildTable(flights);
        buildGlobe(flights);
      },
    });

    // 2) AIRPORT COORDS (Add or expand to include all airport codes)
    const airportCoords = {
      ICN: [37.4602, 126.4407],
      ATL: [33.6367, -84.4281],
      MIA: [25.7954, -80.2901],
      ORD: [41.9742, -87.9073],
      LGA: [40.7769, -73.8740],
      DCA: [38.8512, -77.0402],
      JFK: [40.6413, -73.7781],
      SFO: [37.6213, -122.3790],
      LAX: [33.9416, -118.4085],
      BOS: [42.3656, -71.0096],
      PIT: [40.4914, -80.2329],
      MSP: [44.8848, -93.2223],
      HNL: [21.3245, -157.9251],
      EWR: [40.6895, -74.1745],
      CLT: [35.2144, -80.9473],
      PHX: [33.4342, -112.0116],
      // ...
      // Add any other IATA codes you see in your CSV: NRT, OKA, PVG, SIN, LHR, etc.
      NRT: [35.7732, 140.3874],
      OKA: [26.2048, 127.6460],
      PVG: [31.1443, 121.8083],
      SIN: [1.3644, 103.9915],
      LHR: [51.4700, -0.4543],
      // Continue adding as needed...
    };

    // ========== BUILD SUMMARY ========== //
    function buildSummary(flights) {
      let totalFlights = 0;
      let totalFlightHours = 0;
      let totalCancellations = 0;
      let totalDiversions = 0;

      let sumDepDelayMin = 0;
      let sumArrDelayMin = 0;
      let depDelayCount = 0;
      let arrDelayCount = 0;

      flights.forEach((f) => {
        totalFlights++;

        // Canceled?
        if (String(f.Canceled).toLowerCase() === 'true') {
          totalCancellations++;
        }
        // Diverted?
        if (f['Diverted To'] && f['Diverted To'].trim() !== '') {
          totalDiversions++;
        }

        // Flight hours: "Take off (Actual)" -> "Landing (Actual)" or fallback gate times
        const depActual = f['Take off (Actual)'] || f['Gate Departure (Actual)'];
        const arrActual = f['Landing (Actual)'] || f['Gate Arrival (Actual)'];
        const depDate = new Date(depActual);
        const arrDate = new Date(arrActual);
        if (!isNaN(depDate) && !isNaN(arrDate) && arrDate > depDate) {
          const diffMs = arrDate - depDate;
          totalFlightHours += diffMs / (1000 * 60 * 60);
        }

        // Dep Delay: Gate Departure (Actual) - Gate Departure (Scheduled)
        const depSch = new Date(f['Gate Departure (Scheduled)']);
        const depAct = new Date(f['Gate Departure (Actual)']);
        if (!isNaN(depSch) && !isNaN(depAct)) {
          sumDepDelayMin += (depAct - depSch) / (1000 * 60);
          depDelayCount++;
        }

        // Arr Delay: Gate Arrival (Actual) - Gate Arrival (Scheduled)
        const arrSch = new Date(f['Gate Arrival (Scheduled)']);
        const arrAct = new Date(f['Gate Arrival (Actual)']);
        if (!isNaN(arrSch) && !isNaN(arrAct)) {
          sumArrDelayMin += (arrAct - arrSch) / (1000 * 60);
          arrDelayCount++;
        }
      });

      const avgDepDelay =
        depDelayCount > 0
          ? (sumDepDelayMin / depDelayCount).toFixed(1)
          : 0;
      const avgArrDelay =
        arrDelayCount > 0
          ? (sumArrDelayMin / arrDelayCount).toFixed(1)
          : 0;

      document.getElementById('total-flights').textContent = totalFlights;
      document.getElementById('total-flight-hours').textContent =
        totalFlightHours.toFixed(1);
      document.getElementById('avg-dep-delay').textContent = avgDepDelay;
      document.getElementById('avg-arr-delay').textContent = avgArrDelay;
      document.getElementById('total-cancellations').textContent =
        totalCancellations;
      document.getElementById('total-diversions').textContent =
        totalDiversions;
    }

    // ========== BUILD MONTHLY FLIGHT CHART ========== //
    function buildChart(flights) {
      const flightsByMonth = {};
      flights.forEach((f) => {
        const d = new Date(f.Date);
        if (!isNaN(d)) {
          const year = d.getFullYear();
          const month = d.getMonth() + 1; // 1-based
          const ym = `${year}-${String(month).padStart(2, '0')}`;
          flightsByMonth[ym] = (flightsByMonth[ym] || 0) + 1;
        }
      });

      const labels = Object.keys(flightsByMonth).sort();
      const data = labels.map((k) => flightsByMonth[k]);

      const ctx = document.getElementById('flightsByMonth').getContext('2d');
      new Chart(ctx, {
        type: 'bar',
        data: {
          labels,
          datasets: [
            {
              label: 'Flights per Month',
              data,
              backgroundColor: 'rgba(54, 162, 235, 0.6)',
            },
          ],
        },
        options: {
          responsive: true,
          scales: {
            x: {
              title: { display: true, text: 'Month (YYYY-MM)' },
            },
            y: {
              title: { display: true, text: 'Flights' },
              beginAtZero: true,
            },
          },
        },
      });
    }

    // ========== BUILD TABLE (INCLUDE ALL COLUMNS) ========== //
    function buildTable(flights) {
      const tbody = document.querySelector('#flightsTable tbody');
      flights.forEach((f) => {
        const tr = document.createElement('tr');

        // The order of columns matches your CSV header order:
        const cols = [
          f.Date,
          f.Airline,
          f.Flight,
          f.From,
          f.To,
          f['Dep Terminal'],
          f['Dep Gate'],
          f['Arr Terminal'],
          f['Arr Gate'],
          f.Canceled,
          f['Diverted To'],
          f['Gate Departure (Scheduled)'],
          f['Gate Departure (Actual)'],
          f['Take off (Scheduled)'],
          f['Take off (Actual)'],
          f['Landing (Scheduled)'],
          f['Landing (Actual)'],
          f['Gate Arrival (Scheduled)'],
          f['Gate Arrival (Actual)'],
          f['Aircraft Type Name'],
          f['Tail Number'],
          f.PNR,
          f.Seat,
          f['Seat Type'],
          f['Cabin Class'],
          f['Flight Reason'],
          f.Notes,
          f['Flight Flighty ID'],
          f['Airline Flighty ID'],
          f['Departure Airport Flighty ID'],
          f['Arrival Airport Flighty ID'],
          f['Diverted To Airport Flighty ID'],
          f['Aircraft Type Flighty ID'],
        ];

        cols.forEach((val) => {
          const td = document.createElement('td');
          td.textContent = val || '';
          tr.appendChild(td);
        });

        tbody.appendChild(tr);
      });
    }

    // ========== BUILD 3D GLOBE WITH CESIUM ==========
    function buildGlobe(flights) {
      // For better imagery, sign up for a free Cesium Ion token if you like:
      // Cesium.Ion.defaultAccessToken = 'YOUR_TOKEN_HERE';

      const viewer = new Cesium.Viewer('cesiumContainer', {
        animation: false,
        timeline: false,
        baseLayerPicker: true,
        geocoder: false,
      });

      // Optionally hide Cesium ion credits
      viewer.cesiumWidget.creditContainer.style.display = 'none';

      flights.forEach((f) => {
        const from = f.From;
        const to = f.To;
        if (!from || !to) return;

        const fromCoords = airportCoords[from];
        const toCoords = airportCoords[to];
        if (!fromCoords || !toCoords) return; // skip if coords missing

        // Cesium expects [lon, lat] order
        const [lat1, lon1] = fromCoords;
        const [lat2, lon2] = toCoords;

        viewer.entities.add({
          polyline: {
            positions: Cesium.Cartesian3.fromDegreesArray([
              lon1,
              lat1,
              lon2,
              lat2,
            ]),
            width: 2,
            material: Cesium.Color.fromCssColorString('#007aff').withAlpha(0.7),
          },
        });
      });

      // Fly the camera to a global view
      viewer.scene.camera.flyHome(2.0);
    }
  });
</script>
