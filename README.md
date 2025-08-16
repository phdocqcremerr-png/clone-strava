# clone-strava
gps vélo
<!-- /index.html -->
<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Mon Strava Clone pour Vélo</title>

  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" integrity="sha256-p4NxAoJBhIIN+hmNHrzRCf9tD/miZyoHS5obTRR9BMY=" crossorigin=""/>

  <style>
    body{font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,Helvetica,Arial,sans-serif;margin:0;padding:15px;background:#f4f4f4;display:flex;flex-direction:column;align-items:center}
    #app-container{max-width:900px;width:100%;background:#fff;padding:20px;border-radius:8px;box-shadow:0 2px 10px rgba(0,0,0,.1)}
    h1{color:#333;text-align:center;margin:0 0 10px}
    #map{height:480px;width:100%;margin-top:12px;border-radius:6px;border:1px solid #ddd}
    #controls,#stats{margin-top:16px;display:flex;flex-wrap:wrap;gap:12px;justify-content:center}
    button,input[type="file"],label.toggle{padding:10px 14px;font-size:16px;border-radius:6px;border:1px solid #ddd;background:#fff;cursor:pointer;transition:background-color .3s,color .3s}
    button:hover{background:#f0f0f0}
    button:disabled{cursor:not-allowed;opacity:.5}
    #start-tracking{background:#28a745;color:#fff;border-color:#28a745}
    #stop-tracking{background:#dc3545;color:#fff;border-color:#dc3545}
    #connect-cadence{background:#007bff;color:#fff;border-color:#007bff}
    #reset{background:#6c757d;color:#fff;border-color:#6c757d}
    #stats p{font-size:18px;color:#555;background:#f9f9f9;padding:10px;border-radius:6px;flex:1;text-align:center;min-width:150px}
    #stats span{font-weight:700;color:#000}
    #alerts{margin-top:8px;color:#b45400;text-align:center;min-height:1.2em}
    .row{display:flex;flex-wrap:wrap;gap:10px;justify-content:center;align-items:center}
    .toggle{display:inline-flex;gap:8px;align-items:center;user-select:none}
    .toggle input{transform:scale(1.2)}
  </style>
</head>
<body>
  <div id="app-container">
    <h1>Mon Strava Clone Vélo</h1>

    <div id="map"></div>

    <div id="stats" class="row">
      <p>Distance: <span id="distance">0.00</span> km</p>
      <p>D+: <span id="elevation">0</span> m</p>
      <p>Cadence: <span id="cadence">0</span> RPM</p>
      <p>Fix GPS: <span id="gps-qual">—</span></p>
    </div>

    <div id="controls" class="row">
      <input type="file" id="gpx-file" accept=".gpx" title="Importer un fichier GPX">
      <button id="start-tracking">Démarrer le suivi GPS</button>
      <button id="stop-tracking" disabled>Arrêter le suivi</button>
      <button id="connect-cadence">Connecter capteur</button>
      <button id="reset">Réinitialiser</button>
      <label class="toggle" title="Si l'altitude GPS est indisponible, interroger un service d'élévation public">
        <input type="checkbox" id="use-elev-api">
        Utiliser API d'élévation (fallback)
      </label>
    </div>

    <div id="alerts" aria-live="polite"></div>
  </div>

  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js" integrity="sha256-20nQCchB9co0qIjJZRGuk2/Z9VM+kNiyxNV1lvTlZBo=" crossorigin=""></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/leaflet-gpx/1.7.0/gpx.min.js"></script>

  <script>
    // Pourquoi HTTPS: Web Bluetooth & Geolocation haute précision exigent une origine sécurisée.

    // --------- Carte ----------
    const map = L.map('map').setView([49.93, 5.71], 13);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',{
      attribution:'&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>'
    }).addTo(map);

    // --------- État global ----------
    let gpxLayer = null;
    let userPolyline = null;

    let positions = [];           // [[lat,lon], ...] pour le tracé live
    let watchId = null;

    let totalDistanceM = 0;       // cumul en mètres
    let elevationGainM = 0;       // D+ en mètres

    let lastPos = null;           // {lat,lon}
    let lastAlt = null;           // en mètres (null si inconnu)

    const elevCache = new Map();  // 'lat,lon' -> altitude
    let lastElevQueryTs = 0;

    // --------- UI refs ----------
    const distanceEl = document.getElementById('distance');
    const elevationEl = document.getElementById('elevation');
    const cadenceEl = document.getElementById('cadence');
    const gpsQualEl = document.getElementById('gps-qual');
    const startBtn = document.getElementById('start-tracking');
    const stopBtn = document.getElementById('stop-tracking');
    const resetBtn = document.getElementById('reset');
    const gpxInput = document.getElementById('gpx-file');
    const connectBtn = document.getElementById('connect-cadence');
    const useElevApiChk = document.getElementById('use-elev-api');
    const alertsEl = document.getElementById('alerts');

    // --------- Utils ----------
    const km = m => (m/1000).toFixed(2);
    const clamp = (v,min,max) => Math.max(min,Math.min(max,v));
    const keyLL = (lat,lon) => `${lat.toFixed(5)},${lon.toFixed(5)}`;

    function setAlert(msg){ alertsEl.textContent = msg || ''; }
    function clearMap(){
      if (gpxLayer) { map.removeLayer(gpxLayer); gpxLayer = null; }
      if (userPolyline) { map.removeLayer(userPolyline); userPolyline = null; }
      positions = [];
    }
    function resetStats(){
      totalDistanceM = 0;
      elevationGainM = 0;
      lastPos = null;
      lastAlt = null;
      distanceEl.textContent = '0.00';
      elevationEl.textContent = '0';
      gpsQualEl.textContent = '—';
    }
    function updateStatsUI(){
      distanceEl.textContent = km(totalDistanceM);
      elevationEl.textContent = Math.round(elevationGainM).toString();
    }
    function positiveGain(prev, curr){
      const diff = curr - prev;
      return diff > 0 ? diff : 0;
    }

    // Simple anti-rafale pour API d'élévation publique (courtoisie)
    async function fetchElevation(lat, lon){
      const k = keyLL(lat, lon);
      if (elevCache.has(k)) return elevCache.get(k);

      const now = Date.now();
      const delay = 250; // éviter de spammer les services publics
      const wait = Math.max(0, lastElevQueryTs + delay - now);
      if (wait) await new Promise(r=>setTimeout(r, wait));
      lastElevQueryTs = Date.now();

      // Service public libre ; sujet aux quotas. Utilisé seulement en secours.
      const url = `https://api.open-elevation.com/api/v1/lookup?locations=${lat},${lon}`;

      try{
        const res = await fetch(url);
        if(!res.ok) throw new Error(`HTTP ${res.status}`);
        const data = await res.json();
        const ele = data?.results?.[0]?.elevation;
        if (typeof ele === 'number'){
          elevCache.set(k, ele);
          return ele;
        }
      }catch(_e){
        // Ne pas briser le flux si l'API échoue
      }
      return null;
    }

    // --------- GPX import ----------
    gpxInput.addEventListener('change', e=>{
      const file = e.target.files?.[0];
      if(!file) return;

      if (watchId){
        setAlert("Arrête le suivi GPS avant d'importer un GPX.");
        gpxInput.value = '';
        return;
      }

      clearMap();
      resetStats();
      setAlert('');

      const reader = new FileReader();
      reader.onload = ev=>{
        gpxLayer = new L.GPX(ev.target.result, {
          async: true,
          marker_options:{
            startIconUrl:'https://cdnjs.cloudflare.com/ajax/libs/leaflet-gpx/1.7.0/pin-icon-start.png',
            endIconUrl:'https://cdnjs.cloudflare.com/ajax/libs/leaflet-gpx/1.7.0/pin-icon-end.png',
            shadowUrl:'https://cdnjs.cloudflare.com/ajax/libs/leaflet-gpx/1.7.0/pin-shadow.png'
          }
        })
        .on('loaded', ev2=>{
          map.fitBounds(ev2.target.getBounds());
          const pts = ev2.target.get_points();
          // Calcul local distance & D+
          let dist = 0, gain = 0;
          for(let i=1;i<pts.length;i++){
            const p1 = pts[i-1], p2 = pts[i];
            dist += L.latLng(p1.lat, p1.lon).distanceTo(L.latLng(p2.lat, p2.lon));
            const dEle = (p2.ele ?? 0) - (p1.ele ?? 0);
            if (dEle > 0) gain += dEle;
          }
          totalDistanceM = dist;
          elevationGainM = gain;
          updateStatsUI();
          setAlert(`GPX chargé: ${pts.length} points`);
        })
        .on('error', ()=> setAlert("Erreur lors du chargement du GPX."))
        .addTo(map);
      };
      reader.readAsText(file);
    });

    // --------- Suivi GPS live ----------
    startBtn.addEventListener('click', ()=>{
      if (!navigator.geolocation){
        setAlert("La géolocalisation n'est pas supportée par ce navigateur.");
        return;
      }
      clearMap();
      resetStats();
      setAlert('Suivi en cours…');
      gpxInput.disabled = true;
      startBtn.disabled = true;
      stopBtn.disabled = false;

      watchId = navigator.geolocation.watchPosition(
        async pos=>{
          const { latitude:lat, longitude:lon, accuracy, altitude, altitudeAccuracy } = pos.coords;

          gpsQualEl.textContent = `${Math.round(accuracy)} m${altitudeAccuracy!=null?` / Alt ±${Math.round(altitudeAccuracy)} m`:''}`;

          const ll = [lat,lon];
          positions.push(ll);

          if (!userPolyline){
            userPolyline = L.polyline(positions, { weight: 3 }).addTo(map);
          }else{
            userPolyline.setLatLngs(positions);
          }
          if (!lastPos) map.setView(ll, clamp(map.getZoom(), 14, 18));

          // Distance
          if (lastPos){
            const inc = L.latLng(lastPos.lat, lastPos.lon).distanceTo(L.latLng(lat, lon));
            if (inc > 0.2) totalDistanceM += inc; // filtre micro-bruit
          }
          lastPos = {lat, lon};

          // Altitude
          let currAlt = (typeof altitude === 'number') ? altitude : null;
          if (currAlt == null && useElevApiChk.checked){
            currAlt = await fetchElevation(lat, lon);
          }
          if (currAlt != null){
            if (lastAlt != null){
              const incGain = positiveGain(lastAlt, currAlt);
              if (incGain < 50) elevationGainM += incGain; // filtre pics aberrants GPS
            }
            lastAlt = currAlt;
          }

          updateStatsUI();
        },
        err=>{
          setAlert(`Erreur GPS: ${err.message}`);
          stopTracking();
        },
        { enableHighAccuracy:true, maximumAge:0, timeout:10000 }
      );
    });

    function stopTracking(){
      if (watchId){
        navigator.geolocation.clearWatch(watchId);
        watchId = null;
      }
      startBtn.disabled = false;
      stopBtn.disabled = true;
      gpxInput.disabled = false;
      setAlert('Suivi arrêté.');
    }

    stopBtn.addEventListener('click', stopTracking);
    resetBtn.addEventListener('click', ()=>{
      if (watchId) stopTracking();
      clearMap();
      resetStats();
      setAlert('');
      gpxInput.value = '';
    });

    // --------- Cadence BLE (CSC) ----------
    connectBtn.addEventListener('click', async ()=>{
      if (!navigator.bluetooth){
        setAlert("Web Bluetooth non supporté par ce navigateur.");
        return;
      }
      setAlert('Connexion au capteur… (assure-toi d\'être en HTTPS)');
      try{
        const device = await navigator.bluetooth.requestDevice({
          filters:[{ services: ['cycling_speed_and_cadence'] }]
        });
        const server = await device.gatt.connect();
        const service = await server.getPrimaryService('cycling_speed_and_cadence');
        const ch = await service.getCharacteristic('csc_measurement'); // 0x2A5B
        await ch.startNotifications();
        ch.addEventListener('characteristicvaluechanged', handleCadenceChanged);
        setAlert('Capteur connecté. Lecture de la cadence…');
      }catch(e){
        setAlert(`Erreur Bluetooth: ${e.message}`);
      }
    });

    // Variables & constantes CSC
    let lastCrankRevs = null;     // u16 cumulatif (rollover)
    let lastCrankTime = null;     // u16 ticks (1/1024 s)
    const U16 = 0x10000;          // 65536

    function handleCadenceChanged(event){
      const dv = event.target.value;
      const flags = dv.getUint8(0);
      let idx = 1;

      const wheelPresent = (flags & 0x01) !== 0;
      const crankPresent = (flags & 0x02) !== 0;

      if (wheelPresent) idx += 6; // CWR (u32) + LWET (u16)

      if (crankPresent){
        const currRevs = dv.getUint16(idx, true);
        const currTime = dv.getUint16(idx+2, true); // 1/1024 s

        if (lastCrankRevs != null && lastCrankTime != null){
          // Gestion rollover u16
          const dRevs = (currRevs - lastCrankRevs + U16) % U16;
          let dTicks = (currTime - lastCrankTime + U16) % U16;

          if (dTicks > 0 && dRevs >= 0){
            const dt = dTicks / 1024;           // en s
            const rpm = (dRevs / dt) * 60;
            if (isFinite(rpm) && rpm >= 0 && rpm < 250){ // filtre valeurs folles
              cadenceEl.textContent = Math.round(rpm).toString();
            }
          }
        }
        lastCrankRevs = currRevs;
        lastCrankTime = currTime;
      }
    }

    // Conseils d'installation (pour l'utilisateur final, pas besoin de modifier le code):
    // - Héberge cette page via HTTPS (ex: GitHub Pages) pour activer Web Bluetooth & haute précision GPS.
    // - Coche "API d'élévation (fallback)" si l'altitude GPS est manquante/instable.
  </script>
</body>
</html>
