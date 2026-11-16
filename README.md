<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>Live Satellite Orbit Tracker</title>
<style>
  :root {
    box-sizing: border-box;
    padding-top: env(safe-area-inset-top, 0px);
    padding-bottom: env(safe-area-inset-bottom, 0px);
    --bg: #03060c;
    --card: rgba(12, 18, 30, 0.85);
    --fg: #eef2f8;
    --mut: #94a3b8;
    --line: rgba(148, 163, 184, 0.24);
    --acc: #ff8a4c;
  }
  html, body {
    height: 100%;
    margin: 0;
    background: var(--bg);
    overflow: hidden;
    font: 13.5px/1.4 system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
    color: var(--fg);
    user-select: none;
    -webkit-user-select: none;
  }
  #stage {
    position: relative;
    width: 100%;
    height: 100%;
  }
  canvas {
    display: block;
  }
  /* Floating 2D Satellite Name Labels Container */
  #labels {
    position: absolute;
     inset: 0;
    pointer-events: none;
    overflow: hidden;
    z-index: 2;
  }
  .sat-label {
    position: absolute;
    top: 0;
    left: 0;
    pointer-events: auto;
    cursor: pointer;
    padding: 2px 7px;
    border-radius: 999px;
    background: rgba(7, 12, 22, 0.78);
    border: 1px solid rgba(255, 255, 255, 0.22);
    color: #f1f5f9;
    font-size: 11px;
    font-weight: 500;
    letter-spacing: 0.01em;
    white-space: nowrap;
    backdrop-filter: blur(4px);
    -webkit-backdrop-filter: blur(4px);
    transition: border-color 0.15s, background 0.15s;
    will-change: transform;
  }
  .sat-label:hover {
    background: rgba(24, 36, 60, 0.92);
    border-color: rgba(255, 255, 255, 0.55);
  }
  .sat-label.selected {
    background: rgba(255, 138, 76, 0.25);
    border-color: #ffd54a;
    color: #ffd54a;
    font-weight: 700;
    z-index: 3;
  }
  /* HUD Overlays */
  .ui {
    position: absolute;
    left: 12px;
    right: 12px;
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    align-items: center;
    pointer-events: none;
    z-index: 5;
  }
  .ui > * {
    pointer-events: auto;
  }
  #top {
    top: 12px;
  }
  #bot {
    bottom: 12px;
  }
  .pill, button, input {
    font: inherit;
    font-size: 13px;
    background: var(--card);
    color: var(--fg);
    border: 1px solid var(--line);
    border-radius: 999px;
    padding: 6px 14px;
    backdrop-filter: blur(8px);
    -webkit-backdrop-filter: blur(8px);
    box-shadow: 0 4px 16px rgba(0, 0, 0, 0.35);
  }
  .live-dot {
    display: inline-block;
    width: 7px;
    height: 7px;
    border-radius: 50%;
    background: #3ccf8e;
    margin-right: 6px;
    box-shadow: 0 0 8px #3ccf8e;
  }
  button {
    cursor: pointer;
    transition: background 0.15s, border-color 0.15s;
  }
  button:hover {
    border-color: rgba(255, 255, 255, 0.45);
    background: rgba(26, 36, 56, 0.92);
  }
  input {
    width: 210px;
    outline: none;
  }
  input:focus {
    border-color: var(--acc);
  }
  .card {
    flex: 1 1 100%;
    background: var(--card);
    border: 1px solid var(--line);
    border-radius: 14px;
    padding: 10px 16px;
    backdrop-filter: blur(8px);
    -webkit-backdrop-filter: blur(8px);
    box-shadow: 0 4px 20px rgba(0, 0, 0, 0.45);
  }
  .sm {
    color: var(--mut);
    font-size: 12px;
    margin-top: 2px;
  }
</style>
</head>
<body>
<div id="stage">
  <!-- Floating 2D Satellite Name Labels -->
  <div id="labels"></div>

  <!-- Top Live Tracking Bar -->
  <div class="ui" id="top">
    <span class="pill"><span class="live-dot"></span><span id="clock">--</span></span>
    <span class="pill" id="count">--</span>
    <button id="loc">My location</button>
    <input id="q" placeholder="Search satellite or lat,lon…">
  </div>

  <!-- Bottom Telemetry & Status Bar -->
  <div class="ui" id="bot">
    <div class="card">
      <div>
        <b id="iname">Drag to rotate, scroll/pinch to zoom in for satellite &amp; city names, tap a dot for telemetry.</b>
        <span id="idet" style="margin-left:6px;color:#ffd54a;"></span>
      </div>
      <div class="sm" id="status">Initializing live satellite tracker…</div>
    </div>
  </div>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/controls/OrbitControls.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/satellite.js/5.0.0/satellite.min.js"></script>
<script>
const RE = 6371, MU = 398600.4418, D2R = Math.PI / 180, T0 = Date.now(), $ = id => document.getElementById(id);
const TLE_URL = 'https://celestrak.org/NORAD/elements/gp.php?GROUP=active&FORMAT=tle';
const gm = t => ((280.46061837 + 360.98564736629 * (t / 864e5 + 2440587.5 - 2451545)) % 360) * D2R;

// Setup WebGL Renderer & Scene
const stage = $('stage'), labelsContainer = $('labels');
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
stage.insertBefore(renderer.domElement, stage.firstChild);

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(45, 1, 0.005, 600);
camera.position.set(0, 1.15, 3.4);

// Build Equirectangular Hybrid Satellite Imagery + Country/State/City Labels Texture
function createSatelliteMapTexture() {
  const W = 4096, H = 2048;
  const c = document.createElement('canvas');
  c.width = W;
  c.height = H;
  const ctx = c.getContext('2d');

  // Deep ocean initial background while satellite imagery tiles stream in
  const grad = ctx.createLinearGradient(0, 0, 0, H);
  grad.addColorStop(0, '#091b32');
  grad.addColorStop(0.5, '#061529');
  grad.addColorStop(1, '#091b32');
  ctx.fillStyle = grad;
  ctx.fillRect(0, 0, W, H);

  const tex = new THREE.CanvasTexture(c);
  tex.anisotropy = renderer.capabilities.getMaxAnisotropy();

  // Web Mercator tile latitude bounds -> Equirectangular Y pixel coordinate
  const tileLat = (y, n) => {
    const k = Math.PI - 2 * Math.PI * y / n;
    return (180 / Math.PI) * Math.atan(0.5 * (Math.exp(k) - Math.exp(-k)));
  };

  // Fallback high-res Blue Marble base before/under Esri tiles
  const baseImg = new Image();
  baseImg.crossOrigin = 'anonymous';
  baseImg.onload = () => {
    ctx.drawImage(baseImg, 0, 0, W, H);
    drawStaticGeoLabels(ctx, W, H);
    tex.needsUpdate = true;
    loadEsriHybridTiles();
  };
  baseImg.onerror = () => {
    drawStaticGeoLabels(ctx, W, H);
    tex.needsUpdate = true;
    loadEsriHybridTiles();
  };
  baseImg.src = 'https://unpkg.com/three-globe@2.31.1/example/img/earth-blue-marble.jpg';

  function loadEsriHybridTiles() {
    const Z = 4, N = 1 << Z; // 16x16 tiles at Zoom 4 = 4096x4096 source resolution
    const satTiles = Array.from({ length: N }, () => Array(N).fill(null));
    const refTiles = Array.from({ length: N }, () => Array(N).fill(null));

    let pendingRedraw = false;
    const scheduleRedraw = () => {
      if (pendingRedraw) return;
      pendingRedraw = true;
      requestAnimationFrame(() => {
        pendingRedraw = false;
        // Draw loaded satellite imagery tiles projected into equirectangular space
        for (let y = 0; y < N; y++) {
          const latTop = tileLat(y, N), latBot = tileLat(y + 1, N);
          const py0 = (90 - latTop) / 180 * H;
          const py1 = (90 - latBot) / 180 * H;
          const ph = Math.max(1, py1 - py0);
          for (let x = 0; x < N; x++) {
            const px0 = (x / N) * W;
            const pw = W / N;
            if (satTiles[y][x]) {
              ctx.drawImage(satTiles[y][x], px0, py0, pw + 0.6, ph + 0.6);
            }
          }
        }
        // Draw Esri World Boundaries and Places (country/state borders & city/country/state names) on top
        for (let y = 0; y < N; y++) {
          const latTop = tileLat(y, N), latBot = tileLat(y + 1, N);
          const py0 = (90 - latTop) / 180 * H;
          const py1 = (90 - latBot) / 180 * H;
          const ph = Math.max(1, py1 - py0);
          for (let x = 0; x < N; x++) {
            const px0 = (x / N) * W;
            const pw = W / N;
            if (refTiles[y][x]) {
              ctx.drawImage(refTiles[y][x], px0, py0, pw + 0.6, ph + 0.6);
            }
          }
        }
        // Re-apply crisp major state & city labels so zooming into regions shows clear place names
        drawStaticGeoLabels(ctx, W, H);
        tex.needsUpdate = true;
      });
    };

    for (let y = 0; y < N; y++) {
      for (let x = 0; x < N; x++) {
        const sImg = new Image();
        sImg.crossOrigin = 'anonymous';
        sImg.onload = () => { satTiles[y][x] = sImg; scheduleRedraw(); };
        sImg.src = `https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/${Z}/${y}/${x}`;

        const rImg = new Image();
        rImg.crossOrigin = 'anonymous';
        rImg.onload = () => { refTiles[y][x] = rImg; scheduleRedraw(); };
        rImg.src = `https://services.arcgisonline.com/ArcGIS/rest/services/Reference/World_Boundaries_and_Places/MapServer/tile/${Z}/${y}/${x}`;
      }
    }
  }

  return tex;
}

// Supplemental crisp Country, State/Province, and Major City labels on the globe texture
function drawStaticGeoLabels(ctx, W, H) {
  const X = lon => ((lon + 180) / 360) * W;
  const Y = lat => ((90 - lat) / 180) * H;

  ctx.save();
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';

  // States / Provinces (subtle uppercase labels)
  const states = [
    ['CALIFORNIA', 36.7, -119.4], ['TEXAS', 31.5, -99.3], ['FLORIDA', 28.6, -81.7],
    ['NEW YORK', 42.9, -75.5], ['ILLINOIS', 40.0, -89.2], ['IOWA', 42.0, -93.5],
    ['WASHINGTON', 47.4, -120.5], ['COLORADO', 39.0, -105.5], ['ALASKA', 64.2, -152.5],
    ['ONTARIO', 50.0, -86.0], ['QUEBEC', 52.0, -71.5], ['BRITISH COLUMBIA', 54.0, -125.0],
    ['ALBERTA', 54.5, -115.0], ['NEW SOUTH WALES', -32.5, 147.0], ['WESTERN AUSTRALIA', -25.5, 122.0],
    ['QUEENSLAND', -23.0, 144.0], ['SÃO PAULO', -22.2, -48.8], ['AMAZONAS', -4.2, -64.5],
    ['SICHUAN', 30.6, 102.7], ['XINJIANG', 41.1, 85.2], ['MAHARASHTRA', 19.6, 75.3]
  ];
  ctx.font = '600 11px system-ui, sans-serif';
  ctx.lineWidth = 2.5;
  ctx.strokeStyle = 'rgba(0, 0, 0, 0.75)';
  ctx.fillStyle = 'rgba(226, 232, 240, 0.88)';
  for (const [name, lat, lon] of states) {
    const px = X(lon), py = Y(lat);
    ctx.strokeText(name, px, py);
    ctx.fillText(name, px, py);
  }

  // Major World & Regional Cities (dot + crisp label)
  const cities = [
    ['New York', 40.71, -74.01], ['Los Angeles', 34.05, -118.24], ['Chicago', 41.88, -87.63],
    ['Cedar Rapids', 41.98, -91.66], ['Houston', 29.76, -95.37], ['Miami', 25.76, -80.19],
    ['Seattle', 47.61, -122.33], ['Denver', 39.74, -104.99], ['Toronto', 43.65, -79.38],
    ['Vancouver', 49.28, -123.12], ['Mexico City', 19.43, -99.13], ['Bogotá', 4.71, -74.07],
    ['Lima', -12.05, -77.04], ['São Paulo', -23.55, -46.63], ['Buenos Aires', -34.60, -58.38],
    ['Santiago', -33.45, -70.67], ['London', 51.51, -0.13], ['Paris', 48.86, 2.35],
    ['Madrid', 40.42, -3.70], ['Berlin', 52.52, 13.40], ['Rome', 41.90, 12.50],
    ['Warsaw', 52.23, 21.01], ['Istanbul', 41.01, 28.98], ['Moscow', 55.76, 37.62],
    ['Cairo', 30.04, 31.24], ['Lagos', 6.52, 3.38], ['Nairobi', -1.29, 36.82],
    ['Johannesburg', -26.20, 28.05], ['Dubai', 25.20, 55.27], ['Riyadh', 24.71, 46.68],
    ['Mumbai', 19.08, 72.88], ['New Delhi', 28.61, 77.21], ['Bangkok', 13.76, 100.50],
    ['Singapore', 1.35, 103.82], ['Jakarta', -6.21, 106.85], ['Beijing', 39.90, 116.41],
    ['Shanghai', 31.23, 121.47], ['Hong Kong', 22.32, 114.17], ['Seoul', 37.57, 126.98],
    ['Tokyo', 35.68, 139.65], ['Manila', 14.60, 120.98], ['Sydney', -33.87, 151.21],
    ['Melbourne', -37.81, 144.96], ['Perth', -31.95, 115.86], ['Auckland', -36.85, 174.76],
    ['Honolulu', 21.31, -157.86], ['Anchorage', 61.22, -149.90]
  ];
  ctx.font = '500 11px system-ui, sans-serif';
  ctx.textAlign = 'left';
  for (const [name, lat, lon] of cities) {
    const px = X(lon), py = Y(lat);
    ctx.beginPath();
    ctx.arc(px, py, 2.2, 0, Math.PI * 2);
    ctx.fillStyle = '#ffd54a';
    ctx.fill();
    ctx.strokeStyle = 'rgba(0,0,0,0.85)';
    ctx.lineWidth = 2.5;
    ctx.strokeText(name, px + 4, py);
    ctx.fillStyle = '#ffffff';
    ctx.fillText(name, px + 4, py);
  }
  ctx.restore();
}

const earth = new THREE.Mesh(
  new THREE.SphereGeometry(1, 96, 64),
  new THREE.MeshLambertMaterial({ map: createSatelliteMapTexture() })
);
scene.add(earth);

// Subtle atmospheric limb glow
scene.add(new THREE.Mesh(
  new THREE.SphereGeometry(1.022, 64, 48),
  new THREE.MeshBasicMaterial({ color: 0x4c9aff, transparent: true, opacity: 0.11, side: THREE.BackSide })
));

// Balanced lighting so night-side satellite imagery & city/country names remain clearly readable
const sun = new THREE.DirectionalLight(0xffffff, 0.72);
const ambient = new THREE.AmbientLight(0xdbeafe, 0.68);
scene.add(sun, ambient);

// Background starfield
{
  const a = new Float32Array(2700);
  for (let i = 0; i < a.length; i += 3) {
    const v = new THREE.Vector3(Math.random() - 0.5, Math.random() - 0.5, Math.random() - 0.5)
      .normalize()
      .multiplyScalar(300);
    a.set([v.x, v.y, v.z], i);
  }
  const g = new THREE.BufferGeometry();
  g.setAttribute('position', new THREE.BufferAttribute(a, 3));
  scene.add(new THREE.Points(g, new THREE.PointsMaterial({ size: 1.4, sizeAttenuation: false, color: 0x9fb3d1 })));
}

function dotTex(ring) {
  const c = document.createElement('canvas');
  c.width = c.height = 64;
  const g = c.getContext('2d');
  g.beginPath();
  g.arc(32, 32, ring ? 26 : 30, 0, 7);
  if (ring) {
    g.lineWidth = 6;
    g.strokeStyle = '#fff';
    g.stroke();
  } else {
    g.fillStyle = '#fff';
    g.fill();
  }
  return new THREE.CanvasTexture(c);
}

const ctl = new THREE.OrbitControls(camera, renderer.domElement);
ctl.enableDamping = true;
ctl.enablePan = false;
ctl.minDistance = 1.02;
ctl.maxDistance = 16;

// User geolocation red pin
const mark = new THREE.Mesh(
  new THREE.SphereGeometry(0.006, 16, 12),
  new THREE.MeshBasicMaterial({ color: 0xff3b3b })
);
mark.visible = false;
scene.add(mark);

// Selected satellite highlight ring
const selPt = new THREE.Points(
  new THREE.BufferGeometry().setAttribute('position', new THREE.BufferAttribute(new Float32Array(3), 3)),
  new THREE.PointsMaterial({
    size: 22,
    sizeAttenuation: false,
    map: dotTex(true),
    transparent: true,
    alphaTest: 0.3,
    color: 0xffd54a,
    depthTest: false
  })
);
selPt.visible = false;
selPt.frustumCulled = false;
scene.add(selPt);

let sats = [], pos, pts, geo, sel = null, line = null, cur = 0, fly = null, isLive = false, pathAt = 0;
const CH = 1000;

// Fallback illustrative satellites if CelesTrak is unreachable
function mk(name, alt, inc, raan, ph, color) {
  const a = RE + alt, n = Math.sqrt(MU / a ** 3), i = inc * D2R, o = raan * D2R, u0 = ph * D2R;
  return {
    name,
    color,
    period: 2 * Math.PI / n,
    vel: () => Math.sqrt(MU / a),
    at(t) {
      const u = u0 + n * (t - T0) / 1e3, cu = Math.cos(u), su = Math.sin(u), ci = Math.cos(i), co = Math.cos(o), so = Math.sin(o),
        x = a * (co * cu - so * ci * su), y = a * (so * cu + co * ci * su), z = a * Math.sin(i) * su,
        g = gm(t), cg = Math.cos(g), sg = Math.sin(g);
      return [(x * cg + y * sg) / RE, z / RE, (x * sg - y * cg) / RE];
    }
  };
}

function demoSats() {
  const s = [
    mk('ISS (ZARYA)', 420, 51.6, 40, 0, '#ff8a4c'),
    mk('HST (HUBBLE)', 540, 28.5, 120, 90, '#ff8a4c'),
    mk('CSS (TIANHE)', 390, 41.5, 200, 200, '#ff8a4c')
  ];
  for (let p = 0; p < 12; p++)
    for (let k = 0; k < 6; k++)
      s.push(mk('STARLINK-' + (1000 + p * 6 + k + 1), 550, 53, p * 30, k * 60 + p * 7, '#4c9aff'));
  for (let p = 0; p < 6; p++)
    for (let k = 0; k < 4; k++)
      s.push(mk('GPS BIIR-' + (p * 4 + k + 1), 20200, 55, p * 60, k * 90 + p * 15, '#3ccf8e'));
  for (let i = 0; i < 12; i++)
    s.push(mk('GOES-' + (i + 1), 35786, 0.1, 0, i * 30, '#b57cff'));
  for (let i = 0; i < 10; i++)
    s.push(mk('NOAA-' + (i + 15), 700, 98.2, i * 36, i * 50, '#26c6c6'));
  return s;
}

function fromTLE(txt) {
  const L = txt.split(/\r?\n/).map(s => s.trim()).filter(Boolean), out = [];
  for (let i = 0; i < L.length - 1 && out.length < 20000; i++) {
    if (L[i].startsWith('1 ') && L[i + 1].startsWith('2 ')) {
      const nm = i > 0 && !/^[12] /.test(L[i - 1]) ? L[i - 1] : 'SAT ' + (out.length + 1);
      let rec;
      try { rec = satellite.twoline2satrec(L[i], L[i + 1]); } catch (e) { continue; }
      i++;
      if (!rec || !rec.no) continue;
      const a = Math.cbrt(MU / ((rec.no / 60) ** 2)), alt = a - RE;
      const color = /^(ISS \(ZARYA\)|CSS \(TIANHE\)|HST)$/.test(nm)
        ? '#ff8a4c'
        : alt < 2000 ? '#4c9aff' : alt < 30000 ? '#3ccf8e' : (alt > 34000 && alt < 38000) ? '#b57cff' : '#26c6c6';
      out.push({
        name: nm,
        color,
        period: 2 * Math.PI / rec.no * 60,
        vel(t) {
          const p = satellite.propagate(rec, new Date(t));
          return p && p.velocity ? Math.hypot(p.velocity.x, p.velocity.y, p.velocity.z) : 0;
        },
        at(t) {
          const d = new Date(t), p = satellite.propagate(rec, d);
          if (!p || !p.position) return null;
          const e = satellite.eciToEcf(p.position, satellite.gstime(d));
          return [e.x / RE, e.z / RE, -e.y / RE];
        }
      });
    }
  }
  return out;
}

function setSats(s, msg) {
  sats = s;
  if (pts) {
    scene.remove(pts);
    geo.dispose();
  }
  pos = new Float32Array(s.length * 3);
  const col = new Float32Array(s.length * 3);
  s.forEach((x, i) => {
    const c = new THREE.Color(x.color);
    col.set([c.r, c.g, c.b], i * 3);
  });
  geo = new THREE.BufferGeometry();
  geo.setAttribute('position', new THREE.BufferAttribute(pos, 3));
  geo.setAttribute('color', new THREE.BufferAttribute(col, 3));
  geo.boundingSphere = new THREE.Sphere(new THREE.Vector3(), 60);
  pts = new THREE.Points(
    geo,
    new THREE.PointsMaterial({
      size: 4.5,
      sizeAttenuation: false,
      vertexColors: true,
      map: dotTex(false),
      transparent: true,
      alphaTest: 0.4
    })
  );
  pts.frustumCulled = false;
  scene.add(pts);
  cur = 0;
  pick0(null);
  const nowT = Date.now();
  for (let i = 0; i < s.length; i += CH) step(i, Math.min(s.length, i + CH), nowT);
  $('status').textContent = msg;
}

function step(a, b, t) {
  for (let i = a; i < b; i++) {
    const p = sats[i].at(t);
    if (p) pos.set(p, i * 3);
    else pos.set([0, 0, 0], i * 3);
  }
  geo.attributes.position.needsUpdate = true;
}

function sunDir(t) {
  const d = t / 864e5 + 2440587.5 - 2451545,
    L = (280.46 + 0.9856474 * d) * D2R,
    g = (357.528 + 0.9856003 * d) * D2R,
    lam = L + (1.915 * Math.sin(g) + 0.02 * Math.sin(2 * g)) * D2R,
    e = 23.439 * D2R,
    dec = Math.asin(Math.sin(e) * Math.sin(lam)),
    ra = Math.atan2(Math.cos(e) * Math.sin(lam), Math.cos(lam)),
    lo = ra - gm(t);
  return new THREE.Vector3(Math.cos(dec) * Math.cos(lo), Math.sin(dec), -Math.cos(dec) * Math.sin(lo));
}

function pick0(s) {
  sel = s;
  if (line) {
    scene.remove(line);
    line.geometry.dispose();
    line = null;
  }
  selPt.visible = !!s;
  pathAt = 0;
  if (!s) {
    $('iname').textContent = 'Drag to rotate, scroll/pinch to zoom in for satellite & city names, tap a dot for telemetry.';
    $('idet').textContent = '';
  } else {
    $('iname').textContent = s.name;
  }
}

function path(t) {
  if (line) {
    scene.remove(line);
    line.geometry.dispose();
  }
  const N = 160, a = new Float32Array(N * 3);
  for (let k = 0; k < N; k++) {
    a.set(sel.at(t + (k / (N - 1)) * sel.period * 1e3) || [0, 0, 0], k * 3);
  }
  const g = new THREE.BufferGeometry();
  g.setAttribute('position', new THREE.BufferAttribute(a, 3));
  line = new THREE.Line(g, new THREE.LineBasicMaterial({ color: 0xffd54a, transparent: true, opacity: 0.75 }));
  line.frustumCulled = false;
  scene.add(line);
}

function flyTo(v, dist) {
  fly = {
    a: camera.position.clone().normalize(),
    b: v.clone().normalize(),
    t0: performance.now(),
    d0: camera.position.length(),
    d1: dist
  };
}

function ll(lat, lon) {
  const la = lat * D2R, lo = lon * D2R;
  return new THREE.Vector3(Math.cos(la) * Math.cos(lo), Math.sin(la), -Math.cos(la) * Math.sin(lo));
}

function goLoc(lat, lon) {
  const v = ll(lat, lon);
  mark.position.copy(v.clone().multiplyScalar(1.002));
  mark.visible = true;
  flyTo(v, 1.15);
}

$('loc').onclick = () => {
  if (!navigator.geolocation) {
    $('status').textContent = 'Geolocation is unavailable. Type lat,lon in the search box instead.';
    return;
  }
  navigator.geolocation.getCurrentPosition(
    p => goLoc(p.coords.latitude, p.coords.longitude),
    () => ($('status').textContent = 'Location permission blocked. Type lat,lon in the search box (e.g. 41.98,-91.66).'),
    { timeout: 8000 }
  );
};

$('q').addEventListener('keydown', e => {
  if (e.key !== 'Enter') return;
  const v = e.target.value.trim(),
    m = v.match(/^(-?\d+(?:\.\d+)?)\s*,\s*(-?\d+(?:\.\d+)?)$/);
  if (m) {
    goLoc(+m[1], +m[2]);
    return;
  }
  const lv = v.toLowerCase(),
    s = sats.find(x => x.name.toLowerCase().includes(lv));
  if (s) {
    pick0(s);
    const p = s.at(Date.now());
    if (p) flyTo(new THREE.Vector3(...p), Math.max(1.25, Math.hypot(...p) + 0.35));
  } else {
    $('status').textContent = 'No active satellite matches "' + v + '".';
  }
});

// Raycasting for clicking satellite dots
const ray = new THREE.Raycaster(), el = renderer.domElement;
let dn = null;
el.addEventListener('pointerdown', e => (dn = [e.clientX, e.clientY]));
el.addEventListener('pointerup', e => {
  if (!dn || Math.hypot(e.clientX - dn[0], e.clientY - dn[1]) > 5) return;
  const r = el.getBoundingClientRect();
  ray.setFromCamera(new THREE.Vector2(((e.clientX - r.left) / r.width) * 2 - 1, -((e.clientY - r.top) / r.height) * 2 + 1), camera);
  ray.params.Points.threshold = 0.014 * (camera.position.length() - 0.9) + 0.004;
  const h = ray.intersectObject(pts), eh = ray.intersectObject(earth);
  const ok = h.filter(x => !eh.length || x.distance < eh[0].distance);
  pick0(ok.length ? sats[ok[0].index] : null);
});

// Pool of 2D DOM Label elements for smooth 60fps zoom-in satellite names
const MAX_LABELS = 45;
const labelPool = [];
for (let i = 0; i < MAX_LABELS; i++) {
  const div = document.createElement('div');
  div.className = 'sat-label';
  div.style.display = 'none';
  div.onclick = e => {
    e.stopPropagation();
    if (div._sat) pick0(div._sat);
  };
  labelsContainer.appendChild(div);
  labelPool.push(div);
}

const tmpV = new THREE.Vector3();
const camPos = new THREE.Vector3();

function updateSatelliteLabels(w, h) {
  camPos.copy(camera.position);
  const camDist = camPos.length();
  // Show nearby satellite names when zoomed in closer than 2.35 Earth radii (or always for the selected satellite)
  const showNearby = camDist < 2.35;

  if (!showNearby && !sel) {
    for (let i = 0; i < MAX_LABELS; i++) labelPool[i].style.display = 'none';
    return;
  }

  // Horizon distance from camera to Earth tangent for occlusion check
  const horizonDistSq = Math.max(0, camDist * camDist - 1.0);
  const candidates = [];

  for (let i = 0, n = sats.length; i < n; i++) {
    const s = sats[i];
    const isSelected = s === sel;
    if (!showNearby && !isSelected) continue;

    const idx = i * 3;
    const sx = pos[idx], sy = pos[idx + 1], sz = pos[idx + 2];
    if (sx === 0 && sy === 0 && sz === 0) continue;

    const dx = sx - camPos.x, dy = sy - camPos.y, dz = sz - camPos.z;
    const distSq = dx * dx + dy * dy + dz * dz;

    // Quick distance filter when zoomed in
    if (!isSelected && distSq > 1.35 * (camDist - 0.85)) continue;

    // Check if occluded by the Earth sphere
    const satRadiusSq = sx * sx + sy * sy + sz * sz;
    if (distSq > horizonDistSq + Math.max(0, satRadiusSq - 1.0)) continue;

    tmpV.set(sx, sy, sz).project(camera);
    if (tmpV.z < -1 || tmpV.z > 1) continue;
    if (Math.abs(tmpV.x) > 0.96 || Math.abs(tmpV.y) > 0.94) continue;

    const screenScore = isSelected ? -1e9 : (tmpV.x * tmpV.x + tmpV.y * tmpV.y) + distSq * 0.5;
    candidates.push({
      sat: s,
      x: (tmpV.x * 0.5 + 0.5) * w,
      y: (-tmpV.y * 0.5 + 0.5) * h,
      score: screenScore,
      isSelected
    });
  }

  candidates.sort((a, b) => a.score - b.score);
  const count = Math.min(candidates.length, MAX_LABELS);

  for (let i = 0; i < MAX_LABELS; i++) {
    const div = labelPool[i];
    if (i < count) {
      const c = candidates[i];
      div.style.display = 'block';
      div.style.transform = `translate3d(${Math.round(c.x + 8)}px, ${Math.round(c.y - 10)}px, 0)`;
      if (div._sat !== c.sat) {
        div._sat = c.sat;
        div.textContent = c.sat.name;
      }
      div.classList.toggle('selected', c.isSelected);
    } else {
      div.style.display = 'none';
      div._sat = null;
    }
  }
}

let stageW = 1, stageH = 1;
function size() {
  stageW = stage.clientWidth;
  stageH = stage.clientHeight;
  renderer.setSize(stageW, stageH);
  camera.aspect = stageW / stageH;
  camera.updateProjectionMatrix();
}
addEventListener('resize', size);
size();

let uiAt = 0;
function frame(now) {
  const liveNow = Date.now();
  sun.position.copy(sunDir(liveNow).multiplyScalar(20));

  if (sats.length <= CH) {
    step(0, sats.length, liveNow);
  } else {
    step(cur, Math.min(sats.length, cur + CH), liveNow);
    cur += CH;
    if (cur >= sats.length) cur = 0;
  }

  if (sel) {
    const p = sel.at(liveNow);
    if (p) {
      selPt.geometry.attributes.position.set(p, 0);
      selPt.geometry.attributes.position.needsUpdate = true;
      if (now - pathAt > 3000) {
        path(liveNow);
        pathAt = now;
      }
      if (now - uiAt > 250) {
        uiAt = now;
        const r = Math.hypot(...p);
        $('idet').textContent =
          'Lat ' + (Math.asin(p[1] / r) / D2R).toFixed(2) + '°, ' +
          'Lon ' + (Math.atan2(-p[2], p[0]) / D2R).toFixed(2) + '° · ' +
          'Alt ' + Math.round(r * RE - RE).toLocaleString() + ' km · ' +
          sel.vel(liveNow).toFixed(2) + ' km/s';
      }
    }
  }

  if (fly) {
    const k = Math.min(1, (now - fly.t0) / 1400),
      e = k < 0.5 ? 2 * k * k : 1 - Math.pow(-2 * k + 2, 2) / 2;
    camera.position.copy(
      fly.a.clone().lerp(fly.b, e).normalize().multiplyScalar(fly.d0 + (fly.d1 - fly.d0) * e)
    );
    if (k >= 1) fly = null;
  }

  ctl.rotateSpeed = Math.max(0.06, Math.min(1, (camera.position.length() - 1) * 0.6));
  ctl.update();
  renderer.render(scene, camera);

  updateSatelliteLabels(stageW, stageH);

  $('clock').textContent = new Date(liveNow).toISOString().replace('T', ' ').slice(0, 19) + ' UTC';
  $('count').textContent = sats.length.toLocaleString() + ' satellites';
  requestAnimationFrame(frame);
}

async function live(first) {
  try {
    if (!window.satellite) throw 0;
    let txt = null;
    try {
      const c = JSON.parse(localStorage.getItem('tle') || 'null');
      if (c && Date.now() - c.t < 72e5) txt = c.x;
    } catch (e) {}
    if (!txt) {
      const r = await fetch(TLE_URL);
      if (!r.ok) throw 0;
      txt = await r.text();
      try {
        localStorage.setItem('tle', JSON.stringify({ t: Date.now(), x: txt }));
      } catch (e) {}
    }
    const s = fromTLE(txt);
    if (!s.length) throw 0;
    isLive = true;
    setSats(s, 'Live orbital data from CelesTrak · Zoom in to view satellite, country, state & city names.');
  } catch (e) {
    if (first) {
      $('status').textContent = 'Live CelesTrak feed unreachable from this network — showing simulated live orbits.';
    }
  }
}

setSats(demoSats(), 'Connecting to CelesTrak live satellite catalog…');
requestAnimationFrame(frame);
live(true);
setInterval(() => {
  if (isLive) {
    try { localStorage.removeItem('tle'); } catch (e) {}
    live(false);
  }
}, 72e5);
</script>
</body>
</html>
