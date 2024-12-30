---
layout: page
title: "My Flight Visualizations"
permalink: /flight-visuals/
---

<!-- ======================= Flighty-like, 3D Globe Visualization ======================= -->
<!-- 1) Summary Stats 2) Monthly Chart 3) Table 4) 3D Globe (CesiumJS) -->

<style>
  /* ========== STYLING FOR A CLEAN, “FLIGHTY-LIKE” LOOK ========== */

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
  <p>
    <strong>Average Departure Delay (mins):</strong>
    <span id="avg-dep-delay"></span>
  </p>
  <p>
    <strong>Average Arrival Delay (mins):</strong>
    <span id="avg-arr-delay"></span>
  </p>
  <p>
    <strong>Cancellations:</strong>
    <span id="total-cancellations"></span>
  </p>
  <p>
    <strong>Diversions:</strong>
    <span id="total-diversions"></span>
  </p>
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
        <th>Date</th>
        <th>Airline</th>
        <th>Flight #</th>
        <th>From</th>
        <th>To</th>
        <th>Gate Dep (Sched)</th>
        <th>Gate Dep (Actual)</th>
        <th>Gate Arr (Sched)</th>
        <th>Gate Arr (Actual)</th>
        <th>Canceled</th>
        <th>Diverted To</th>
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
    // ========== 1) LOAD CSV ==========
    const csvPath = "{{ '/assets/data/FlightyExport.csv' | relative_url }}";
    Papa.parse(csvPath, {
      download: true,
      header: true,
      complete: function (results) {
        const flights = results.data.filter((f) => f.Date);
        buildSummary(flights);
        buildChart(flights);
        buildTable(flights);
        buildGlobe(flights);
      },
    });

    // ========== 2) AIRPORT COORDS ==========
    // Add all airports that appear in your From/To columns.
    // Format: "IATA/ICAO code": [latitude, longitude]
    const airportCoords = {
      ICN: [37.4602, 126.4407],
      ATL: [33.6367, -84.4281],
      MIA: [25.7954, -80.2901],
      ORD: [41.9742, -87.9073],
      LGA: [40.7769, -73.874],
      DCA: [38.8512, -77.0402],
      JFK: [40.6413, -73.7781],
      SFO: [37.6213, -122.379],
      LAX: [33.9416, -118.4085],
      /* Add more as needed */
    };

    // ========== 3) SUMMARY STATS ==========
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

        // Flight hours:
        // use "Take off (Actual)" -> "Landing (Actual)" if available, else Gate times
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

    // ========== 4) MONTHLY FLIGHT CHART ==========
    function buildChart(flights) {
      const flightsByMonth = {};
      flights.forEach((f) => {
        const d = new Date(f.Date);
        if (!isNaN(d)) {
          const y = d.getFullYear();
          const m = d.getMonth() + 1; // 1-based
          const ym = y + '-' + String(m).padStart(2, '0');
          flightsByMonth[ym] = (flightsByMonth[ym] || 0) + 1;
        }
      });

      const labels = Object.keys(flightsByMonth).sort();
      const data = labels.map((k) => flightsByMonth[k]);

      const ctx = document
        .getElementById('flightsByMonth')
        .getContext('2d');
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

    // ========== 5) FLIGHTS TABLE ==========
    function buildTable(flights) {
      const tbody = document.querySelector('#flightsTable tbody');
      flights.forEach((f) => {
        const tr = document.createElement('tr');
        // Columns:
        const cols = [
          f.Date,
          f.Airline,
          f.Flight,
          f.From,
          f.To,
          f['Gate Departure (Scheduled)'],
          f['Gate Departure (Actual)'],
          f['Gate Arrival (Scheduled)'],
          f['Gate Arrival (Actual)'],
          f.Canceled,
          f['Diverted To'],
        ];
        cols.forEach((val) => {
          const td = document.createElement('td');
          td.textContent = val || '';
          tr.appendChild(td);
        });
        tbody.appendChild(tr);
      });
    }

    // ========== 6) 3D GLOBE (CESIUM) ==========
    function buildGlobe(flights) {
      // For full features, get a free Cesium Ion token: https://cesium.com/ion/
      // Then do: Cesium.Ion.defaultAccessToken = 'YOUR_TOKEN_HERE';

      // Create the viewer in #cesiumContainer
      const viewer = new Cesium.Viewer('cesiumContainer', {
        animation: false,
        timeline: false,
        baseLayerPicker: true,
        geocoder: false,
      });

      // Optionally, remove the Cesium ion word
      viewer.cesiumWidget.creditContainer.style.display = 'none';

      flights.forEach((f) => {
        const fromCode = f.From;
        const toCode = f.To;
        if (!fromCode || !toCode) {
          return;
        }

        const fromCoords = airportCoords[fromCode];
        const toCoords = airportCoords[toCode];
        if (!fromCoords || !toCoords) {
          return; // skip if coords missing
        }

        // [lat, lon] => [longitude, latitude] for Cesium
        const [lat1, lon1] = fromCoords;
        const [lat2, lon2] = toCoords;

        // Create a route as a polyline in 3D
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

      // Optional: fly to the entire globe
      viewer.scene.camera.flyHome(2.0);
    }
  });
</script>
