<!DOCTYPE html>
<html lang="de">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>TomTom Routenplaner</title>

  <script src="https://api.tomtom.com/maps-sdk-for-web/cdn/6.x/6.20.0/maps/maps-web.min.js"></script>
  <script src="https://api.tomtom.com/maps-sdk-for-web/cdn/6.x/6.20.0/services/services-web.min.js"></script>
  <link href="https://api.tomtom.com/maps-sdk-for-web/cdn/6.x/6.20.0/maps/maps.css" rel="stylesheet" />
  <link href="https://cdnjs.cloudflare.com/ajax/libs/materialize/1.0.0/css/materialize.min.css" rel="stylesheet">
  <link href="https://fonts.googleapis.com/icon?family=Material+Icons" rel="stylesheet">

  <style>
    body {
      margin: 0;
      background-color: #eceff1;
      font-family: 'Roboto', sans-serif;
    }
    #map {
      width: 100%;
      height: 60vh;
    }
    .container {
      padding-top: 20px;
    }
    .card-panel {
      padding: 20px;
      border-radius: 10px;
    }
    .btn-large {
      width: 100%;
      margin-top: 15px;
    }
    .info {
      margin-top: 20px;
      padding: 15px;
      background-color: #607d8b;
      color: white;
      text-align: center;
      font-weight: bold;
      border-radius: 8px;
    }
  </style>
</head>

<body>

  <div class="container">
    <div class="card-panel white z-depth-3">
      <h5 class="center-align">Routenplaner</h5>

      <div class="input-field">
        <input type="text" id="to" placeholder="Zielort eingeben">
        <label for="to">Ziel</label>
      </div>

      <button class="btn-large blue" onclick="calculateRouteFromCurrentLocation()">
        <i class="material-icons left">navigation</i> Route von meinem Standort
      </button>
    </div>
  </div>

  <div id="map"></div>
  <div class="info" id="routeInfo">Bitte Ziel eingeben und Route berechnen!</div>

  <script src="https://cdnjs.cloudflare.com/ajax/libs/materialize/1.0.0/js/materialize.min.js"></script>

  <script>
    const apiKey = 'MyFx4h2uQZPsRAW3Y2HOYn2yQgwfecza';

    const map = tt.map({
      key: apiKey,
      container: 'map',
      center: [13.4050, 52.5200], // Start in Berlin
      zoom: 10
    });

    let routeLayer;

    async function geocode(query) {
      const url = `https://api.tomtom.com/search/2/geocode/${encodeURIComponent(query)}.json?key=${apiKey}`;
      const res = await fetch(url);
      const data = await res.json();
      return data.results[0]?.position;
    }

    async function calculateRouteFromCurrentLocation() {
      const toText = document.getElementById('to').value;

      if (!toText) {
        M.toast({html: 'Bitte ein Ziel eingeben!'});
        return;
      }

      if (!navigator.geolocation) {
        alert("Geolocation wird von deinem Browser nicht unterstützt.");
        return;
      }

      navigator.geolocation.getCurrentPosition(async position => {
        const from = {
          lat: position.coords.latitude,
          lon: position.coords.longitude
        };

        const to = await geocode(toText);

        if (!to) {
          M.toast({html: 'Ziel konnte nicht gefunden werden.'});
          return;
        }

        map.flyTo({ center: [from.lon, from.lat], zoom: 12 });
        calculateRoute(from, to);
      }, error => {
        console.error(error);
        alert("Standort konnte nicht ermittelt werden.");
      }, {
        enableHighAccuracy: true,
        timeout: 10000,
        maximumAge: 0
      });
    }

    async function calculateRoute(from, to) {
      const routeUrl = `https://api.tomtom.com/routing/1/calculateRoute/${from.lat},${from.lon}:${to.lat},${to.lon}/json?key=${apiKey}&travelMode=car`;
      const routeRes = await fetch(routeUrl);
      const routeData = await routeRes.json();
      const points = routeData.routes[0].legs[0].points;

      const geoJson = {
        type: 'Feature',
        geometry: {
          type: 'LineString',
          coordinates: points.map(p => [p.longitude, p.latitude])
        }
      };

      if (routeLayer) {
        map.removeLayer('route');
        map.removeSource('route');
      }

      map.addSource('route', { type: 'geojson', data: geoJson });
      map.addLayer({
        id: 'route',
        type: 'line',
        source: 'route',
        paint: {
          'line-color': '#0078A8',
          'line-width': 5
        }
      });
      routeLayer = true;

      const summary = routeData.routes[0].summary;
      const lengthInKm = (summary.lengthInMeters / 1000).toFixed(2);
      const travelTimeInMin = Math.round(summary.travelTimeInSeconds / 60);

      document.getElementById('routeInfo').innerText = `Entfernung: ${lengthInKm} km • Fahrzeit: ${travelTimeInMin} Minuten`;
    }
  </script>

</body>
</html>
