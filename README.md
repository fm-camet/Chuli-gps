<!-- index.html -->
<meta http-equiv="refresh" content="0; url=chuli-gps.html"><!DOCTYPE html><html lang="es">
<head>
  <meta charset="UTF-8">
  <title>Chuli GPS Optimizado</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
  <style>
    body { font-family: sans-serif; background: #f4f4f4; padding: 1em; }
    h1 { text-align: center; color: #333; }
    input, button {
      padding: 10px; margin: 5px 0; width: 100%; font-size: 16px;
    }
    .btn { background: #4CAF50; color: white; border: none; cursor: pointer; }
    .btn:hover { background: #45a049; }
    #map { height: 400px; margin-top: 20px; border-radius: 10px; }
    .item { background: #fff; padding: 10px; margin: 5px 0; border-radius: 6px; }
  </style>
</head>
<body>
  <h1>📍 Chuli GPS Optimizado</h1>  <input type="text" id="direccion" placeholder="Ej: Av. Colón 1234, Mar del Plata" />
  <button class="btn" onclick="agregarDireccion()">Agregar dirección</button>
  <button class="btn" onclick="generarRutaOptimizada()">Ruta Optimizada desde tu ubicación</button>  <div id="listaDirecciones"></div>
  <div id="map"></div>  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>  <script>
    let direcciones = [];
    let mapa = L.map('map').setView([-38.0, -57.55], 12);
    let capaRuta;

    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '© OpenStreetMap contributors'
    }).addTo(mapa);

    function agregarDireccion() {
      const input = document.getElementById("direccion");
      if (input.value.trim() === "") return;
      direcciones.push({ direccion: input.value });
      input.value = "";
      mostrarDirecciones();
    }

    function mostrarDirecciones() {
      const cont = document.getElementById("listaDirecciones");
      cont.innerHTML = "";
      direcciones.forEach((d, i) => {
        cont.innerHTML += `<div class="item">${i + 1}. ${d.direccion}</div>`;
      });
    }

    async function geocodificar(direccion) {
      const url = `https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(direccion)}`;
      const res = await fetch(url);
      const data = await res.json();
      if (data.length === 0) return null;
      return [parseFloat(data[0].lon), parseFloat(data[0].lat)];
    }

    async function generarRutaOptimizada() {
      if (direcciones.length < 1) {
        alert("Agregá al menos una dirección");
        return;
      }

      navigator.geolocation.getCurrentPosition(async (pos) => {
        const ubicacionUsuario = [pos.coords.longitude, pos.coords.latitude];
        const puntos = [ubicacionUsuario];

        for (let d of direcciones) {
          const coord = await geocodificar(d.direccion);
          if (!coord) {
            alert("No se encontró: " + d.direccion);
            return;
          }
          puntos.push(coord);
        }

        // Generar estructura para la API de optimización
        let jobs = puntos.slice(1).map((p, i) => ({ id: i + 1, location: p }));

        const optimizationData = {
          jobs: jobs,
          vehicles: [
            {
              id: 1,
              start: ubicacionUsuario,
              end: ubicacionUsuario
            }
          ]
        };

        const res = await fetch("https://api.openrouteservice.org/optimization", {
          method: "POST",
          headers: {
            "Authorization": "TU_API_KEY_AQUI",
            "Content-Type": "application/json"
          },
          body: JSON.stringify(optimizationData)
        });

        if (!res.ok) {
          console.error("Error optimizando:", await res.text());
          alert("No se pudo optimizar la ruta.");
          return;
        }

        const opt = await res.json();
        const orden = opt.routes[0].steps.map(s => {
          if (s.type === "job") return jobs.find(j => j.id === s.id).location;
          return ubicacionUsuario;
        });

        // Llamar a la API directions con orden correcto
        const res2 = await fetch("https://api.openrouteservice.org/v2/directions/driving-car/geojson", {
          method: "POST",
          headers: {
            "Authorization": "TU_API_KEY_AQUI",
            "Content-Type": "application/json"
          },
          body: JSON.stringify({ coordinates: orden })
        });

        const geojson = await res2.json();
        if (capaRuta) mapa.removeLayer(capaRuta);
        capaRuta = L.geoJSON(geojson, { style: { color: "blue", weight: 4 } }).addTo(mapa);
        mapa.fitBounds(capaRuta.getBounds());

      }, () => alert("No se pudo obtener tu ubicación."));
    }
  </script></body>
</html>
