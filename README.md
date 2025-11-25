# Build-a-Map
Custom map creator
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Interactive Key Account Map</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"/>
  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
  <!-- XLSX for reading Excel / CSV -->
  <script src="https://cdn.jsdelivr.net/npm/xlsx@0.18.5/dist/xlsx.full.min.js"></script>
  <style>
    :root { --print-scale: 1; }

    html, body, #mapid { height: 100%; margin: 0; }

    .emoji-icon {
      font-size: 20px;
      line-height: 1;
    }
    .emoji-icon img {
      width: 20px;
      height: 20px;
      display: block;
      border-radius: 50%;
    }

    .popup-box {
      width: 180px;
      background: white;
      border: 2px solid #333;
      border-radius: 6px;
      padding: 6px;
      text-align: center;
    }

    /* Floating, draggable & resizable tools panel */
    .tools-panel {
      position: absolute;
      top: 12px; left: 12px;
      min-width: 260px; min-height: 140px;
      max-width: 640px; max-height: 80vh;
      background: #fff;
      border: 1px solid #bbb;
      border-radius: 8px;
      box-shadow: 0 6px 18px rgba(0,0,0,0.18);
      font: 12px/1.25 system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif;
      display: flex; flex-direction: column;
      z-index: 1000;
      user-select: none;
    }
    .tools-header {
      display:flex; align-items:center; justify-content:space-between; gap:6px;
      padding: 6px 8px;
      background: #f6f7f8;
      border-bottom: 1px solid #ddd;
      border-radius: 8px 8px 0 0;
      cursor: move;
    }
    .tools-title { font-weight: 600; font-size: 12px; }
    .tools-min-btn {
      border:1px solid #aaa;
      background:#fff;
      padding:1px 6px;
      border-radius:4px;
      cursor:pointer;
      font-size:11px;
    }
    .tools-body { padding: 8px; overflow: auto; }
    .row {
      display:flex;
      gap:6px;
      align-items:center;
      flex-wrap: wrap;
      margin-bottom: 6px;
    }
    .row label { display:flex; gap:4px; align-items:center; }
    .swatch {
      border: 1px solid #aaa;
      background: #fff;
      padding: 2px 6px;
      border-radius: 3px;
      cursor: pointer;
      font-size: 14px;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    .swatch.active { border: 2px solid #000; }

    .swatch-color {
      width: 14px;
      height: 14px;
      border-radius: 50%;
      border: 1px solid #555;
      display: block;
    }

    .btn {
      border:1px solid #aaa;
      background:#fff;
      padding:2px 6px;
      border-radius:3px;
      cursor:pointer;
      font-size:11px;
    }
    .btn:active { transform: translateY(1px); }
    .btn-icon {
      width: 26px;
      text-align:center;
    }
    .hint { font-size:11px; color:#333; }
    .resizer {
      position: absolute; width: 14px; height: 14px; bottom: 4px; right: 4px;
      border-right: 2px solid #aaa; border-bottom: 2px solid #aaa; cursor: nwse-resize; opacity: 0.8;
    }

    #scaleInput {
      width: 60px;
      font-size: 11px;
      text-align: center;
    }

    .location-input {
      flex: 1;
      min-width: 130px;
      padding: 2px 4px;
      font-size: 11px;
      border: 1px solid #aaa;
      border-radius: 3px;
    }

    .mode-label {
      font-size: 11px;
      display: flex;
      align-items: center;
      gap: 2px;
    }

    /* COMPACT MODE */
    .tools-panel.compact {
      min-width: 2.5in; min-height: 2.5in;
      font-size: 11px;
    }
    .tools-panel.compact .tools-header {
      padding: 5px 7px;
    }
    .tools-panel.compact .tools-min-btn {
      padding: 1px 5px; font-size: 10px;
    }
    .tools-panel.compact .tools-body {
      padding: 6px;
    }
    .tools-panel.compact .row {
      gap: 4px; margin-bottom: 4px;
    }
    .tools-panel.compact .swatch {
      padding: 1px 4px; font-size: 12px;
      border-radius: 3px;
    }
    .tools-panel.compact .btn {
      padding: 1px 4px; font-size: 10px; border-radius: 3px;
    }
    .tools-panel.compact .btn-icon {
      width: 22px;
    }
    .tools-panel.compact #scaleInput {
      width: 52px; font-size: 10px;
    }
    .tools-panel.compact .hint {
      font-size: 10px;
    }
    .tools-panel.compact .resizer {
      width: 12px; height: 12px; bottom: 3px; right: 3px;
    }
    .tools-panel.compact .location-input {
      font-size: 10px;
      padding: 1px 3px;
    }
    .tools-panel.compact .mode-label {
      font-size: 10px;
    }

    /* Context menu */
    .ctx-menu {
      position: fixed; z-index: 2000; display: none;
      background: #fff; border: 1px solid #bbb; border-radius: 6px;
      box-shadow: 0 4px 12px rgba(0,0,0,0.15); overflow: hidden;
      min-width: 180px; font: 14px system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif;
    }
    .ctx-item { padding: 8px 12px; cursor: pointer; user-select: none; white-space: nowrap; }
    .ctx-item:hover { background: #f3f3f3; }
    .ctx-item.disabled { color:#888; cursor: not-allowed; }

    /* Confirmation overlay for clearing markers */
    .confirm-overlay {
      position: fixed;
      inset: 0;
      background: rgba(0,0,0,0.35);
      display: none;
      align-items: center;
      justify-content: center;
      z-index: 3000;
    }
    .confirm-box {
      background: #fff;
      padding: 16px 20px;
      border-radius: 8px;
      box-shadow: 0 6px 18px rgba(0,0,0,0.25);
      max-width: 320px;
      text-align: center;
      font: 14px/1.4 system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif;
    }
    .confirm-message {
      margin-bottom: 12px;
      font-weight: 600;
    }
    .confirm-buttons {
      display: flex;
      justify-content: center;
      gap: 10px;
      margin-top: 8px;
    }
    .confirm-btn {
      min-width: 70px;
      padding: 6px 10px;
      border-radius: 4px;
      border: 1px solid #aaa;
      background: #fff;
      cursor: pointer;
      font: inherit;
    }
    .confirm-btn.confirm-yes {
      border-color: #c00;
    }
    .confirm-btn:active {
      transform: translateY(1px);
    }

    /* Globe / round view */
    #mapid.globe-mode {
      width: min(90vmin, 900px);
      height: min(90vmin, 900px);
      border-radius: 50%;
      overflow: hidden;
      margin: 0 auto;
      box-shadow: 0 0 25px rgba(0,0,0,0.4);
    }

    /* Bulk report text */
    .bulk-report {
      white-space: pre-wrap;
      font-size: 11px;
    }

    /* Bulk progress bar */
    .progress-container {
      flex: 1;
      min-width: 120px;
      height: 8px;
      background: #eee;
      border-radius: 4px;
      overflow: hidden;
    }
    .progress-bar {
      height: 100%;
      width: 0%;
      background: #4a90e2;
      transition: width 0.15s linear;
    }

    .auto-retry-select {
      font-size: 11px;
      padding: 2px 4px;
    }

    /* Print-friendly layout */
    @media print {
      .tools-panel, .ctx-menu, .leaflet-control-container, .confirm-overlay { display: none !important; }
      #mapid { height: 100vh !important; transform: scale(var(--print-scale)); transform-origin: top left; }
      html, body { height: 100%; }
    }
  </style>
</head>
<body>
<div id="mapid"></div>

<!-- Floating, draggable, resizable tools panel -->
<div id="toolsPanel" class="tools-panel">
  <div id="toolsHeader" class="tools-header">
    <div class="tools-title">Marker Tools</div>
    <button id="panelToggle" class="tools-min-btn" title="Compact / Expand">◻︎</button>
  </div>
  <div id="toolsBody" class="tools-body">
    <div class="row" id="colorRow">
      <span><b>Color:</b></span>

      <!-- RED -->
      <button class="swatch" data-color="red" title="Red marker">
        <span class="swatch-color" style="background:#e02020;"></span>
      </button>

      <!-- ORANGE -->
      <button class="swatch" data-color="orange" title="Orange marker">
        <span class="swatch-color" style="background:#ff9900;"></span>
      </button>

      <!-- GREEN (default active) -->
      <button class="swatch active" data-color="green" title="Green marker">
        <span class="swatch-color" style="background:#2ecc71;"></span>
      </button>

      <!-- BLUE -->
      <button class="swatch" data-color="blue" title="Blue marker">
        <span class="swatch-color" style="background:#3498db;"></span>
      </button>

      <!-- INDIGO -->
      <button class="swatch" data-color="indigo" title="Indigo marker">
        <span class="swatch-color" style="background:#4b0082;"></span>
      </button>

      <!-- VIOLET -->
      <button class="swatch" data-color="violet" title="Violet marker">
        <span class="swatch-color" style="background:#8a2be2;"></span>
      </button>

      <!-- CUSTOM -->
      <button class="swatch" data-color="custom" title="Custom image marker">
        Custom
      </button>
      <input id="customIconFile" type="file"
             accept="image/png,image/jpeg,image/jpg,image/gif"
             style="display:none;">

      <div style="margin-left:auto; display:flex; gap:6px;">
        <label class="mode-label">
          <input type="radio" name="modeSelect" id="modeIdentify" checked> Identify
        </label>
        <label class="mode-label">
          <input type="radio" name="modeSelect" id="modeNav"> Navigation
        </label>
      </div>
    </div>

    <!-- Location search row -->
    <div class="row">
      <input id="locationSearch" class="location-input" type="text"
             placeholder="e.g. Detroit, MI (Red)">
      <button id="locationAdd" class="btn" title="Geocode and add marker">➕ Add</button>
    </div>

    <!-- Save / Load / Clear / Export / Import -->
    <div class="row">
      <button id="saveMarkers"   class="btn" title="Save markers to this browser">💾 Save</button>
      <button id="loadMarkers"   class="btn" title="Load markers from this browser">📥 Load</button>
      <button id="clearMarkers"  class="btn" title="Clear markers (selection or all)">🧹 Clear</button>
      <button id="exportMarkers" class="btn" title="Download markers as JSON">⬇️ Export</button>
      <label class="btn" for="importFile" title="Import markers JSON">⬆️ Import
        <input id="importFile" type="file" accept="application/json" style="display:none;">
      </label>
    </div>

    <!-- Global View button -->
    <div class="row">
      <button id="globalView" class="btn" title="Toggle round global view">🌍 Global View</button>
    </div>

    <!-- Bulk upload from spreadsheet -->
    <div class="row">
      <span><b>Bulk upload:</b></span>
      <input id="bulkFile" type="file"
             accept=".xlsx,.xls,.csv"
             style="font-size:11px;">
      <span class="hint">Select a color first, then upload City/State list.</span>
    </div>

    <!-- Auto-retry config -->
    <div class="row">
      <span><b>Auto-retries:</b></span>
      <select id="autoRetrySelect" class="auto-retry-select">
        <option value="0">0</option>
        <option value="1" selected>1</option>
        <option value="2">2</option>
        <option value="3">3</option>
      </select>
      <span class="hint">Extra attempts per location</span>
    </div>

    <!-- Marker count -->
    <div class="row">
      <span><b>Markers on map:</b></span>
      <span id="markerCount" class="hint">0</span>
    </div>

    <!-- Bulk status / counter -->
    <div class="row">
      <span><b>Bulk status:</b></span>
      <span id="bulkStatus" class="hint">No bulk file loaded.</span>
    </div>

    <!-- Bulk progress bar -->
    <div class="row">
      <div id="bulkProgressContainer" class="progress-container">
        <div id="bulkProgressBar" class="progress-bar"></div>
      </div>
      <span id="bulkProgressLabel" class="hint">0%</span>
    </div>

    <!-- Retry / Export failed -->
    <div class="row">
      <button id="retryFailed" class="btn" disabled
              title="Retry locations that did not add in the last bulk upload">⟳ Retry Failed</button>
      <button id="exportFailed" class="btn" disabled
              title="Export list of locations that did not add">📄 Export Failed</button>
    </div>

    <!-- Collapsible failed list header -->
    <div class="row" id="failedHeaderRow" style="display:none;">
      <button id="toggleFailed" class="btn" title="Show / hide list of failed locations">
        ▶ Failed locations (0)
      </button>
    </div>

    <!-- Bulk report of locations not added -->
    <div class="row" id="bulkReportRow" style="display:none; max-height:80px; overflow:auto;">
      <span class="bulk-report" id="bulkReport"></span>
    </div>

    <!-- Bulk timestamp -->
    <div class="row">
      <span class="hint" id="bulkTimestamp">Last bulk run: (none)</span>
    </div>

    <!-- Print scale -->
    <div class="row">
      <span><b>Print scale:</b></span>
      <button id="scaleDown" class="btn btn-icon" title="–1%">−</button>
      <input id="scaleInput" type="number" value="100" min="50" max="200" step="1" inputmode="numeric">
      <button id="scaleUp" class="btn btn-icon" title="+1%">+</button>
      <span>%</span>
      <button id="printMap" class="btn" title="Print / Save as PDF">🖨️ Print / PDF</button>
    </div>

    <div class="row">
      <span class="hint">
        In <b>Identify</b> mode: left click + drag on the map to draw a red selection box.<br>
        In <b>Navigation</b> mode: left click + drag to pan the map, click to add markers when a color is selected.<br>
        Clear/Delete removes markers inside the box (or all markers if no box).
      </span>
    </div>
  </div>
  <div id="panelResizer" class="resizer" title="Resize"></div>
</div>

<!-- Right-click context menu -->
<div id="ctxMenu" class="ctx-menu">
  <div id="ctxAdd" class="ctx-item">➕ Add Marker</div>
  <div id="ctxRemove" class="ctx-item">🗑️ Remove Marker</div>
</div>

<!-- Custom confirmation overlay for clearing markers -->
<div id="clearConfirmOverlay" class="confirm-overlay">
  <div class="confirm-box">
    <div class="confirm-message">Are You Sure?</div>
    <div class="confirm-buttons">
      <button id="confirmYes" class="confirm-btn confirm-yes">Yes</button>
      <button id="confirmNo" class="confirm-btn">No</button>
    </div>
  </div>
</div>

<script>
/* --- Map setup --- */
const map = L.map('mapid', {
  center: [20, 0],
  zoom: 2.5,
  worldCopyJump: true,
  zoomSnap: 0.25,
  zoomDelta: 0.25,
  wheelPxPerZoomLevel: 90
});

L.tileLayer('https://tile.jawg.io/730c231f-03a0-4382-980c-5c974f07b8d6/{z}/{x}/{y}{r}.png?access-token=33OVMEOjRZYDyvlqEWEaCECYspGLXatAr9KAZMLNjBnmQsysHwsusaSpaOVxWC9L', {
  attribution: `<a href="https://www.jawg.io" target="_blank">&copy; Jawg</a> - <a href="https://www.openstreetmap.org" target="_blank">&copy; OpenStreetMap</a> contributors`
}).addTo(map);

/* --- Master layer --- */
const allLayer = L.layerGroup().addTo(map);

/* --- Global view helper --- */
function goToGlobalView() {
  const layers = allLayer.getLayers();
  if (layers.length === 0) {
    map.fitWorld({ padding: [30, 30] });
    return;
  }
  const bounds = allLayer.getBounds();
  map.fitBounds(bounds, {
    padding: [30, 30],
    maxZoom: 3
  });
}

/* --- Selection rectangle state (Identify mode: left-click + drag) --- */
let selectionRect = null;
let selectionBounds = null;
let isBoxSelecting = false;
let boxStartPoint = null;

const mapContainer = map.getContainer();

/* --- Mode state: 'identify' or 'nav' --- */
let currentMode = 'identify';
const modeIdentify = document.getElementById('modeIdentify');
const modeNav      = document.getElementById('modeNav');

modeIdentify.addEventListener('change', () => {
  if (modeIdentify.checked) currentMode = 'identify';
});
modeNav.addEventListener('change', () => {
  if (modeNav.checked) {
    currentMode = 'nav';
    if (selectionRect) {
      map.removeLayer(selectionRect);
      selectionRect = null;
    }
    selectionBounds = null;
    map.dragging.enable();
  }
});

/* Start selection ONLY in Identify mode with left button on empty map */
mapContainer.addEventListener('mousedown', (e) => {
  if (currentMode !== 'identify') return;
  if (e.button !== 0) return;

  if (
    e.target.closest('.leaflet-marker-icon') ||
    e.target.closest('.leaflet-control') ||
    e.target.closest('.tools-panel')
  ) return;

  e.preventDefault();
  isBoxSelecting = true;
  boxStartPoint = map.mouseEventToContainerPoint(e);
  selectionBounds = null;

  if (selectionRect) {
    map.removeLayer(selectionRect);
    selectionRect = null;
  }

  map.dragging.disable();
});

mapContainer.addEventListener('mousemove', (e) => {
  if (!isBoxSelecting) return;
  e.preventDefault();

  const currentPoint = map.mouseEventToContainerPoint(e);
  const bounds = L.latLngBounds(
    map.containerPointToLatLng(boxStartPoint),
    map.containerPointToLatLng(currentPoint)
  );

  selectionBounds = bounds;

  if (!selectionRect) {
    selectionRect = L.rectangle(bounds, {
      color: '#ff0000',
      weight: 1,
      fill: false,
      dashArray: '4 2'
    }).addTo(map);
  } else {
    selectionRect.setBounds(bounds);
  }
});

window.addEventListener('mouseup', (e) => {
  if (!isBoxSelecting) return;

  isBoxSelecting = false;
  map.dragging.enable();
  e.preventDefault();

  const endPoint = map.mouseEventToContainerPoint(e);
  const dx = endPoint.x - boxStartPoint.x;
  const dy = endPoint.y - boxStartPoint.y;
  const distSq = dx * dx + dy * dy;
  const dragThresholdSq = 16;

  if (distSq < dragThresholdSq) {
    if (selectionRect) {
      map.removeLayer(selectionRect);
      selectionRect = null;
    }
    selectionBounds = null;
  }
});

/* --- Confirmation overlay helpers --- */
const clearOverlay = document.getElementById('clearConfirmOverlay');
const clearYesBtn  = document.getElementById('confirmYes');
const clearNoBtn   = document.getElementById('confirmNo');

function showClearConfirm() {
  clearOverlay.style.display = 'flex';
}
function hideClearConfirm() {
  clearOverlay.style.display = 'none';
}

/* --- Leaflet Clear Markers button (on the map) --- */
const ClearControl = L.Control.extend({
  options: { position: 'topright' },
  onAdd: function(map) {
    const container = L.DomUtil.create('div', 'leaflet-bar');
    const link = L.DomUtil.create('a', '', container);
    link.href = '#';
    link.title = 'Clear markers (selection or all)';
    link.innerHTML = '🧹';
    L.DomEvent.on(link, 'click', (e) => {
      L.DomEvent.stop(e);
      showClearConfirm();
    });
    return container;
  }
});
map.addControl(new ClearControl());

/* --- Floating tools panel: drag & resize --- */
(() => {
  const panel   = document.getElementById('toolsPanel');
  const header  = document.getElementById('toolsHeader');
  const resizer = document.getElementById('panelResizer');
  const toggle  = document.getElementById('panelToggle');

  let compact = false;
  let dragging = false, resizing = false;
  let startX=0, startY=0, startLeft=0, startTop=0, startW=0, startH=0;

  const MIN_NORMAL = { w: 260, h: 140 };
  const MIN_COMPACT = { w: inchesToPx(2.5), h: inchesToPx(2.5) };
  function inchesToPx(inches){ return Math.round(inches * 96); }

  panel.addEventListener('mousedown', () => panel.style.zIndex = 2000);

  header.addEventListener('mousedown', (e) => {
    dragging = true;
    startX = e.clientX; startY = e.clientY;
    const rect = panel.getBoundingClientRect();
    startLeft = rect.left; startTop = rect.top;
    document.body.style.cursor = 'grabbing';
    e.preventDefault();
  });
  window.addEventListener('mousemove', (e) => {
    if (!dragging) return;
    const dx = e.clientX - startX;
    const dy = e.clientY - startY;
    panel.style.left = Math.max(4, startLeft + dx) + 'px';
    panel.style.top  = Math.max(4, startTop + dy) + 'px';
  });
  window.addEventListener('mouseup', () => {
    dragging = false;
    document.body.style.cursor = '';
  });

  resizer.addEventListener('mousedown', (e) => {
    resizing = true;
    const rect = panel.getBoundingClientRect();
    startW = rect.width; startH = rect.height;
    startX = e.clientX; startY = e.clientY;
    document.body.style.cursor = 'nwse-resize';
    e.preventDefault();
  });
  window.addEventListener('mousemove', (e) => {
    if (!resizing) return;
    const dx = e.clientX - startX;
    const dy = e.clientY - startY;
    const min = compact ? MIN_COMPACT : MIN_NORMAL;
    const newW = Math.min(640, Math.max(min.w, startW + dx));
    const newH = Math.min(window.innerHeight * 0.8, Math.max(min.h, startH + dy));
    panel.style.width = newW + 'px';
    panel.style.height = newH + 'px';
  });
  window.addEventListener('mouseup', () => {
    resizing = false;
    document.body.style.cursor = '';
  });

  toggle.addEventListener('click', () => {
    compact = !compact;
    panel.classList.toggle('compact', compact);
    const rect = panel.getBoundingClientRect();
    const min = compact ? MIN_COMPACT : MIN_NORMAL;
    const targetW = Math.max(min.w, rect.width);
    const targetH = Math.max(min.h, rect.height);
    if (compact) {
      panel.style.width  = Math.max(min.w, Math.min(targetW, Math.round(rect.width * 0.7))) + 'px';
      panel.style.height = Math.max(min.h, Math.min(targetH, Math.round(rect.height * 0.7))) + 'px';
    } else {
      panel.style.width  = Math.min(640, Math.max(MIN_NORMAL.w, Math.round(rect.width * 1.2))) + 'px';
      panel.style.height = Math.min(Math.round(window.innerHeight * 0.8), Math.max(MIN_NORMAL.h, Math.round(rect.height * 1.2))) + 'px';
    }
  });
})();

/* --- Tools panel interactions --- */
let currentColor = 'green';           // null = no color armed for click-to-add
let customIconUrl = null;
let activeSwatch = document.querySelector('#colorRow .swatch.active') || null;

/* Emoji metadata for tooltips/popups */
const colorToEmoji = {
  red:    '🔴',
  orange: '🟠',
  green:  '🟢',
  blue:   '🔵',
  indigo: '🟣',
  violet: '🟣',
  custom: '⭐'
};

/* Handle custom icon upload */
const customIconFile = document.getElementById('customIconFile');
customIconFile.addEventListener('change', (e) => {
  const file = e.target.files[0];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = (ev) => {
    customIconUrl = ev.target.result;  // data URL
    alert('Custom marker icon loaded. Select "Custom" color to use it.');
  };
  reader.readAsDataURL(file);
});

const colorRowEl = document.getElementById('colorRow');

/* Color swatches: toggle on/off, change current color, and optionally recolor selection */
colorRowEl.addEventListener('click', (e) => {
  const sw = e.target.closest('.swatch');
  if (!sw) return;

  if (sw === activeSwatch) {
    sw.classList.remove('active');
    activeSwatch = null;
    currentColor = null;
    return;
  }

  if (activeSwatch) activeSwatch.classList.remove('active');
  sw.classList.add('active');
  activeSwatch = sw;

  const newColor = sw.getAttribute('data-color');
  currentColor = newColor;

  if (newColor === 'custom' && !customIconUrl) {
    customIconFile.click();
  }

  if (selectionBounds && newColor) {
    allLayer.eachLayer(layer => {
      if (!(layer instanceof L.Marker)) return;
      const ll = layer.getLatLng();
      if (!selectionBounds.contains(ll)) return;

      const emojiForMetadata =
        (newColor in colorToEmoji)
          ? colorToEmoji[newColor]
          : (colorToEmoji.green || '🟢');

      const html = getEmojiIconHtml(newColor);
      layer.setIcon(getEmojiIcon(html));
      layer.options.metaColor = (newColor in colorToEmoji) ? newColor : 'custom';
      layer.options.metaEmoji = emojiForMetadata;
    });
  }
});

/* --- Click-to-add markers when a color is armed --- */
map.on('click', (e) => {
  if (!currentColor) return;
  if (currentMode !== 'nav') return;

  const oe = e.originalEvent;
  if (oe) {
    if (
      oe.target.closest('.tools-panel') ||
      oe.target.closest('.leaflet-control') ||
      oe.target.closest('.leaflet-marker-icon')
    ) return;
  }

  const mk = makeMarker(e.latlng, currentColor);
  mk.addTo(allLayer);
});

/* --- Stable print scale controls --- */
const scaleInput = document.getElementById('scaleInput');
const scaleUp    = document.getElementById('scaleUp');
const scaleDown  = document.getElementById('scaleDown');
function clampScale(v){ return Math.max(50, Math.min(200, Math.round(v))); }
scaleInput.addEventListener('wheel', (e)=> e.preventDefault(), { passive:false });
scaleInput.addEventListener('keydown', (e) => {
  const allowed = ['Backspace','Delete','ArrowLeft','ArrowRight','ArrowUp','ArrowDown','Tab','Enter'];
  if (allowed.includes(e.key)) return;
  if (!/^\d$/.test(e.key)) e.preventDefault();
});
scaleInput.addEventListener('input', () => {
  scaleInput.value = clampScale(Number(scaleInput.value) || 100);
});
scaleUp.addEventListener('click', () => scaleInput.value = clampScale(Number(scaleInput.value) + 1));
scaleDown.addEventListener('click', () => scaleInput.value = clampScale(Number(scaleInput.value) - 1));
document.getElementById('printMap').addEventListener('click', () => {
  const val = clampScale(Number(scaleInput.value) || 100);
  document.documentElement.style.setProperty('--print-scale', (val/100).toString());
  window.print();
});

/* --- Save/Load/Export/Import/Clear/GlobalView --- */
const LS_KEY = 'customMarkers_v1';
document.getElementById('saveMarkers').addEventListener('click', saveMarkersToLocalStorage);
document.getElementById('loadMarkers').addEventListener('click', () => loadMarkersFromLocalStorage(true));
document.getElementById('exportMarkers').addEventListener('click', exportMarkersJSON);
document.getElementById('clearMarkers').addEventListener('click', () => showClearConfirm());
document.getElementById('importFile').addEventListener('change', (e) => importMarkersFromFile(e.target.files[0]));

/* Global view toggle -> round globe style */
let isGlobeMode = false;
const globalBtn = document.getElementById('globalView');
globalBtn.addEventListener('click', () => {
  isGlobeMode = !isGlobeMode;
  const mapEl = document.getElementById('mapid');
  if (isGlobeMode) {
    mapEl.classList.add('globe-mode');
    goToGlobalView();
    globalBtn.textContent = '🌍 Globe View (On)';
  } else {
    mapEl.classList.remove('globe-mode');
    globalBtn.textContent = '🌍 Global View';
    map.setView([20, 0], 2.5);
  }
  map.invalidateSize();
});

/* --- Marker helpers --- */
function getEmojiIconHtml(colorOrEmoji) {
  if (colorOrEmoji === 'custom') {
    if (customIconUrl) {
      return `<img src="${customIconUrl}" alt="Custom Marker">`;
    }
    return '⭐';
  }

  const colorMap = {
    red:    '#e02020',
    orange: '#ff9900',
    green:  '#2ecc71',
    blue:   '#3498db',
    indigo: '#4b0082',
    violet: '#8a2be2'
  };

  if (colorMap[colorOrEmoji]) {
    const c = colorMap[colorOrEmoji];
    return `<span class="color-dot" style="
      display:block;
      width:18px;
      height:18px;
      border-radius:50%;
      background:${c};
      border:2px solid #fff;
      box-shadow:0 0 2px rgba(0,0,0,0.4);
    "></span>`;
  }

  const emoji = colorToEmoji[colorOrEmoji] || colorToEmoji.green || '🟢';
  return emoji;
}

function getEmojiIcon(html) {
  return L.divIcon({
    className: 'emoji-icon',
    html: html
  });
}

function formatLatLng(ll) {
  return `${ll.lat.toFixed(5)}, ${ll.lng.toFixed(5)}`;
}

function makeMarker(latlng, colorOrEmoji, name = null) {
  const emojiForMetadata =
    (colorOrEmoji in colorToEmoji)
      ? colorToEmoji[colorOrEmoji]
      : (colorToEmoji.green || '🟢');

  const html = getEmojiIconHtml(colorOrEmoji);

  const mk = L.marker(latlng, {
    icon: getEmojiIcon(html),
    draggable: true
  });

  mk.options.metaColor = (colorOrEmoji in colorToEmoji) ? colorOrEmoji : 'custom';
  mk.options.metaEmoji = emojiForMetadata;
  mk.options.metaName  = name;

  attachMarkerUX(mk, emojiForMetadata);
  return mk;
}

function attachMarkerUX(marker, emoji) {
  marker.on('click', () => {
    const label = marker.options.metaName ? marker.options.metaName : formatLatLng(marker.getLatLng());
    marker.bindPopup(`<div class="popup-box"><b>${emoji}</b><br>${label}</div>`).openPopup();
  });
  marker.on('mouseover', () => {
    const label = marker.options.metaName ? marker.options.metaName : formatLatLng(marker.getLatLng());
    marker.bindTooltip(`${emoji} ${label}`, { direction: 'top' }).openTooltip();
  });
  marker.on('mouseout', () => marker.closeTooltip());

  marker.on('contextmenu', (ev) => {
    if (ev.originalEvent) ev.originalEvent.preventDefault();
    L.DomEvent.stop(ev);
    showCtxMenu(ev.originalEvent.clientX, ev.originalEvent.clientY, { type: 'marker', marker, latlng: ev.latlng });
  });
}

/* --- Context menu logic (right-click) --- */
const ctxMenu   = document.getElementById('ctxMenu');
const ctxAdd    = document.getElementById('ctxAdd');
const ctxRemove = document.getElementById('ctxRemove');
let ctxTarget = null;

function showCtxMenu(x, y, target) {
  ctxTarget = target;
  if (target.type === 'marker') ctxRemove.classList.remove('disabled');
  else                          ctxRemove.classList.add('disabled');

  const menuWidth = 200, menuHeight = 90;
  const vw = window.innerWidth, vh = window.innerHeight;
  const left = Math.min(x, vw - menuWidth - 6);
  const top  = Math.min(y, vh - menuHeight - 6);

  ctxMenu.style.left = left + 'px';
  ctxMenu.style.top  = top + 'px';
  ctxMenu.style.display = 'block';
}
function hideCtxMenu() { ctxMenu.style.display = 'none'; ctxTarget = null; }

document.addEventListener('click', (e) => {
  if (e.target === ctxAdd) {
    if (!ctxTarget) return;
    const colorForMarker = currentColor || 'green';
    const mk = makeMarker(ctxTarget.latlng, colorForMarker);
    mk.addTo(allLayer);
    hideCtxMenu();
  } else if (e.target === ctxRemove) {
    if (!ctxTarget) return;
    if (ctxTarget.type === 'marker' && !ctxRemove.classList.contains('disabled')) {
      allLayer.removeLayer(ctxTarget.marker);
    }
    hideCtxMenu();
  } else {
    if (ctxMenu.style.display === 'block' && !ctxMenu.contains(e.target)) hideCtxMenu();
  }
});
window.addEventListener('scroll', hideCtxMenu);
window.addEventListener('resize', hideCtxMenu);

document.getElementById('mapid').addEventListener('contextmenu', (e) => e.preventDefault());
map.on('contextmenu', (e) => {
  showCtxMenu(e.originalEvent.clientX, e.originalEvent.clientY, { type: 'map', latlng: e.latlng });
});

/* --- Marker count --- */
const markerCountEl = document.getElementById('markerCount');
function updateMarkerCount() {
  let count = 0;
  allLayer.eachLayer(layer => {
    if (layer instanceof L.Marker) count++;
  });
  markerCountEl.textContent = String(count);
}
allLayer.on('layeradd', updateMarkerCount);
allLayer.on('layerremove', updateMarkerCount);

/* --- Persistence --- */
function serializeAllMarkers() {
  const out = [];
  allLayer.eachLayer(layer => {
    if (layer instanceof L.Marker) {
      const { lat, lng } = layer.getLatLng();
      out.push({
        lat,
        lng,
        color: layer.options.metaColor || 'custom',
        emoji: layer.options.metaEmoji || '🟢',
        name: layer.options.metaName || null
      });
    }
  });
  return out;
}

function clearAllMarkersHard() {
  allLayer.clearLayers();
  updateMarkerCount();
}

function recreateFromSerialized(data) {
  clearAllMarkersHard();
  (data || []).forEach(m => {
    const colorOrEmoji = (m.color && (m.color in colorToEmoji)) ? m.color : (m.emoji || '🟢');
    makeMarker([m.lat, m.lng], colorOrEmoji, m.name || null).addTo(allLayer);
  });
  updateMarkerCount();
}

function saveMarkersToLocalStorage() {
  try {
    const payload = serializeAllMarkers();
    localStorage.setItem(LS_KEY, JSON.stringify(payload));
    alert(`Saved ${payload.length} marker(s) to this browser.`);
  } catch (err) { console.error(err); alert('Failed to save markers.'); }
}
function loadMarkersFromLocalStorage(replace = true) {
  try {
    const raw = localStorage.getItem(LS_KEY);
    if (!raw) { alert('No saved markers found.'); return; }
    const payload = JSON.parse(raw);
    if (replace) recreateFromSerialized(payload);
    else payload.forEach(m => makeMarker(
      [m.lat, m.lng],
      (m.color in colorToEmoji) ? m.color : (m.emoji || '🟢'),
      m.name || null
    ).addTo(allLayer));
    updateMarkerCount();
    alert(`Loaded ${payload.length} marker(s).`);
  } catch (err) { console.error(err); alert('Failed to load markers.'); }
}
function exportMarkersJSON() {
  try {
    const payload = serializeAllMarkers();
    const blob = new Blob([JSON.stringify(payload, null, 2)], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url; a.download = 'markers.json';
    document.body.appendChild(a); a.click(); a.remove();
    URL.revokeObjectURL(url);
  } catch (err) { console.error(err); alert('Failed to export markers.'); }
}
function importMarkersFromFile(file) {
  if (!file) return;
  const reader = new FileReader();
  reader.onload = () => {
    try {
      const data = JSON.parse(reader.result);
      if (!Array.isArray(data)) throw new Error('Invalid file format.');
      if (!confirm('Replace current markers with imported markers? Click Cancel to append instead.')) {
        data.forEach(m => makeMarker(
          [m.lat, m.lng],
          (m.color in colorToEmoji) ? m.color : (m.emoji || '🟢'),
          m.name || null
        ).addTo(allLayer));
      } else {
        recreateFromSerialized(data);
      }
      updateMarkerCount();
      alert(`Imported ${data.length} marker(s).`);
    } catch (err) { console.error(err); alert('Failed to import markers.'); }
    finally { document.getElementById('importFile').value = ''; }
  };
  reader.readAsText(file);
}

/* Clear triggered by user (uses selection if present) */
function clearMarkersUserTriggered() {
  if (!selectionBounds) {
    clearAllMarkersHard();
    return;
  }
  const toRemove = [];
  allLayer.eachLayer(layer => {
    if (layer instanceof L.Marker) {
      const ll = layer.getLatLng();
      if (selectionBounds.contains(ll)) {
        toRemove.push(layer);
      }
    }
  });
  toRemove.forEach(m => allLayer.removeLayer(m));
  updateMarkerCount();
}

/* Hook up confirmation buttons */
clearYesBtn.addEventListener('click', () => {
  clearMarkersUserTriggered();
  hideClearConfirm();
});
clearNoBtn.addEventListener('click', () => {
  hideClearConfirm();
});
clearOverlay.addEventListener('click', (e) => {
  if (e.target === clearOverlay) hideClearConfirm();
});

/* Delete key also triggers the clear confirmation (selection or all) */
document.addEventListener('keydown', (e) => {
  if (e.key !== 'Delete') return;
  const tag = (e.target.tagName || '').toLowerCase();
  if (tag === 'input' || tag === 'textarea' || tag === 'select' || e.target.isContentEditable) return;
  showClearConfirm();
});

/* --- Location search & geocoding --- */
const locationInput = document.getElementById('locationSearch');
const locationAddBtn = document.getElementById('locationAdd');

function parseLocationText(raw) {
  let text = raw.trim();
  let color = null;

  const parenMatch = text.match(/\(([^)]+)\)\s*$/);
  if (parenMatch) {
    const colorText = parenMatch[1].trim().toLowerCase();
    if (colorText.startsWith('red'))      color = 'red';
    else if (colorText.startsWith('blue'))   color = 'blue';
    else if (colorText.startsWith('green'))  color = 'green';
    else if (colorText.startsWith('orange')) color = 'orange';
    else if (colorText.startsWith('indigo')) color = 'indigo';
    else if (colorText.startsWith('violet') || colorText.startsWith('purple')) color = 'violet';
    text = text.slice(0, parenMatch.index).trim();
  }

  return { query: text, color };
}

async function geocodePlace(q) {
  const url = 'https://nominatim.openstreetmap.org/search?format=json&limit=1&q=' + encodeURIComponent(q);
  const res = await fetch(url, { headers: { 'Accept': 'application/json' }});
  if (!res.ok) throw new Error('Geocoding request failed');
  const data = await res.json();
  if (!data || !data.length) return null;
  return {
    lat: parseFloat(data[0].lat),
    lon: parseFloat(data[0].lon),
    displayName: data[0].display_name
  };
}

const autoRetrySelect = document.getElementById('autoRetrySelect');
function getAutoRetryCount() {
  const n = parseInt(autoRetrySelect.value, 10);
  if (isNaN(n) || n < 0) return 0;
  return Math.min(3, n);
}

async function geocodeWithRetries(q, retries) {
  for (let attempt = 0; attempt <= retries; attempt++) {
    try {
      const result = await geocodePlace(q);
      if (result) return result;
    } catch (err) {
      console.error('Geocode error (attempt', attempt + 1, 'for', q, '):', err);
    }
  }
  return null;
}

async function handleLocationAdd() {
  const raw = locationInput.value.trim();
  if (!raw) {
    alert('Please enter a location.');
    return;
  }

  const { query, color } = parseLocationText(raw);
  if (!query) {
    alert('Please enter a location name.');
    return;
  }

  locationAddBtn.disabled = true;
  const originalLabel = locationAddBtn.textContent;
  locationAddBtn.textContent = '…';

  try {
    const retries = getAutoRetryCount();
    const result = await geocodeWithRetries(query, retries);
    if (!result) {
      alert('Location not found.');
      return;
    }

    const chosenColor = color || currentColor || 'green';
    const m = makeMarker([result.lat, result.lon], chosenColor, result.displayName || query);
    m.addTo(allLayer);
    map.setView([result.lat, result.lon], 8);
  } catch (err) {
    console.error(err);
    alert('Failed to look up that location.');
  } finally {
    locationAddBtn.disabled = false;
    locationAddBtn.textContent = originalLabel;
  }
}

locationAddBtn.addEventListener('click', handleLocationAdd);
locationInput.addEventListener('keydown', (e) => {
  if (e.key === 'Enter') {
    e.preventDefault();
    handleLocationAdd();
  }
});

/* --- Bulk upload from Excel/CSV with City/State + Retry Failed --- */
const bulkInput = document.getElementById('bulkFile');
const bulkStatusEl = document.getElementById('bulkStatus');
const bulkReportRow = document.getElementById('bulkReportRow');
const bulkReportEl = document.getElementById('bulkReport');
const retryFailedBtn = document.getElementById('retryFailed');
const exportFailedBtn = document.getElementById('exportFailed');
const bulkProgressBar = document.getElementById('bulkProgressBar');
const bulkProgressLabel = document.getElementById('bulkProgressLabel');
const bulkTimestampEl = document.getElementById('bulkTimestamp');
const failedHeaderRow = document.getElementById('failedHeaderRow');
const toggleFailedBtn = document.getElementById('toggleFailed');

let lastFailedLocations = [];
let failedListExpanded = false;

function setBulkProgress(processed, total) {
  if (!total || total <= 0) {
    bulkProgressBar.style.width = '0%';
    bulkProgressLabel.textContent = '0%';
    return;
  }
  const pct = Math.round((processed / total) * 100);
  bulkProgressBar.style.width = pct + '%';
  bulkProgressLabel.textContent = pct + '%';
}

function refreshFailedUI(labelPrefix = 'Locations not added:') {
  const count = lastFailedLocations.length;
  if (count) {
    failedHeaderRow.style.display = 'flex';
    toggleFailedBtn.textContent = (failedListExpanded ? '▼ ' : '▶ ') + `Failed locations (${count})`;
    bulkReportEl.textContent = labelPrefix + '\n' + lastFailedLocations.join('\n');
    bulkReportRow.style.display = failedListExpanded ? 'flex' : 'none';
    retryFailedBtn.disabled = false;
    exportFailedBtn.disabled = false;
  } else {
    failedHeaderRow.style.display = 'none';
    bulkReportRow.style.display = 'none';
    bulkReportEl.textContent = '';
    retryFailedBtn.disabled = true;
    exportFailedBtn.disabled = true;
  }
}

toggleFailedBtn.addEventListener('click', () => {
  failedListExpanded = !failedListExpanded;
  refreshFailedUI(lastFailedLocations.length
    ? bulkReportEl.textContent.split('\n')[0] || 'Locations not added:'
    : 'Locations not added:');
});

bulkInput.addEventListener('change', handleBulkUpload);
retryFailedBtn.addEventListener('click', retryFailedLocations);
exportFailedBtn.addEventListener('click', exportFailedList);

function markBulkTimestamp() {
  const ts = new Date().toLocaleString();
  bulkTimestampEl.textContent = `Last bulk run: ${ts}`;
}

async function handleBulkUpload(e) {
  const file = e.target.files[0];
  if (!file) return;

  if (!currentColor) {
    alert('Please select a marker color first (swatch in the key box).');
    e.target.value = '';
    return;
  }

  const reader = new FileReader();
  reader.onload = async (evt) => {
    try {
      lastFailedLocations = [];
      failedListExpanded = false;
      refreshFailedUI();
      setBulkProgress(0, 1);

      const data = new Uint8Array(evt.target.result);
      const workbook = XLSX.read(data, { type: 'array' });
      const sheetName = workbook.SheetNames[0];
      const sheet = workbook.Sheets[sheetName];
      const rows = XLSX.utils.sheet_to_json(sheet, { header: 1 });

      if (!rows || !rows.length) {
        alert('No rows found in the uploaded file.');
        bulkStatusEl.textContent = 'No rows found in file.';
        setBulkProgress(0, 1);
        return;
      }

      let startIndex = 0;
      let cityCol = 0;
      let stateCol = 1;

      const header = rows[0].map(v => String(v || '').trim().toLowerCase());
      const hasHeaderCity = header.includes('city');
      const hasHeaderState = header.includes('state');

      if (hasHeaderCity || hasHeaderState) {
        cityCol = header.indexOf('city');
        stateCol = header.indexOf('state');
        if (cityCol === -1) cityCol = 0;
        if (stateCol === -1) stateCol = 1;
        startIndex = 1;
      }

      const locations = [];
      for (let i = startIndex; i < rows.length; i++) {
        const row = rows[i];
        if (!row) continue;
        const city = (row[cityCol] || '').toString().trim();
        const state = (row[stateCol] || '').toString().trim();
        if (!city) continue;
        const query = state ? `${city}, ${state}` : city;
        locations.push(query);
      }

      if (!locations.length) {
        alert('No City/State data found. Expected columns named "City" and "State" or the first two columns to be City and State.');
        bulkStatusEl.textContent = 'No valid City/State data found.';
        setBulkProgress(0, 1);
        return;
      }

      const total = locations.length;
      bulkStatusEl.textContent = `Loaded ${total} location(s) from file.`;
      if (!confirm(`Geocode and add ${total} location(s) as ${currentColor} markers?`)) {
        bulkStatusEl.textContent = 'Bulk upload cancelled.';
        setBulkProgress(0, 1);
        return;
      }

      let processed = 0;
      let successCount = 0;
      const failed = [];
      const retries = getAutoRetryCount();

      bulkStatusEl.textContent = `Pending ${total} location(s)...`;
      setBulkProgress(0, total);

      for (const loc of locations) {
        const result = await geocodeWithRetries(loc, retries);
        if (result) {
          const m = makeMarker([result.lat, result.lon], currentColor, result.displayName || loc);
          m.addTo(allLayer);
          successCount++;
        } else {
          console.warn('Not found after retries:', loc);
          failed.push(loc);
        }
        processed++;
        setBulkProgress(processed, total);
        bulkStatusEl.textContent = `Processed ${processed} of ${total} location(s)...`;
      }

      bulkStatusEl.textContent =
        `Bulk upload complete. Added ${successCount} marker(s); ${failed.length} location(s) not added (of ${total}).`;

      lastFailedLocations = failed.slice();
      markBulkTimestamp();
      refreshFailedUI('Locations not added:');

      alert(`Finished bulk upload. Added ${successCount} marker(s). ${failed.length} location(s) not added.`);

      if (successCount) {
        goToGlobalView();
      }
    } catch (err) {
      console.error(err);
      alert('Failed to read or process the uploaded file.');
      bulkStatusEl.textContent = 'Error reading or processing file.';
      setBulkProgress(0, 1);
      // Keep previous failed list if there was one
      refreshFailedUI(
        lastFailedLocations.length
          ? 'Locations not added (from previous run):'
          : 'Locations not added:'
      );
    } finally {
      e.target.value = '';
    }
  };
  reader.readAsArrayBuffer(file);
}

async function retryFailedLocations() {
  if (!lastFailedLocations.length) {
    alert('There are no failed locations to retry.');
    refreshFailedUI();
    return;
  }

  if (!currentColor) {
    alert('Please select a marker color first (swatch in the key box).');
    return;
  }

  const total = lastFailedLocations.length;
  let processed = 0;
  let successCount = 0;
  const stillFailed = [];
  const retries = getAutoRetryCount();

  bulkStatusEl.textContent = `Retrying ${total} failed location(s)...`;
  setBulkProgress(0, total);

  for (const loc of lastFailedLocations) {
    const result = await geocodeWithRetries(loc, retries);
    if (result) {
      const m = makeMarker([result.lat, result.lon], currentColor, result.displayName || loc);
      m.addTo(allLayer);
      successCount++;
    } else {
      console.warn('Still not found after retry:', loc);
      stillFailed.push(loc);
    }
    processed++;
    setBulkProgress(processed, total);
    bulkStatusEl.textContent =
      `Retrying: processed ${processed} of ${total} failed location(s)...`;
  }

  lastFailedLocations = stillFailed.slice();
  markBulkTimestamp();
  refreshFailedUI('Locations still not added after retry:');

  bulkStatusEl.textContent =
    `Retry complete. Added ${successCount} marker(s); ${stillFailed.length} location(s) still not added.`;

  alert(`Retry finished. Added ${successCount} marker(s). ${stillFailed.length} location(s) still not added.`);

  if (successCount) {
    goToGlobalView();
  }
}

function exportFailedList() {
  if (!lastFailedLocations.length) {
    alert('There are no failed locations to export.');
    return;
  }
  const lines = ['Location'];
  lastFailedLocations.forEach(loc => lines.push(`"${loc.replace(/"/g, '""')}"`));
  const csv = lines.join('\n');
  const blob = new Blob([csv], { type: 'text/csv' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'failed_locations.csv';
  document.body.appendChild(a);
  a.click();
  a.remove();
  URL.revokeObjectURL(url);
}

/* --- Load initial markers from markers.json (optional) --- */
fetch('markers.json')
  .then(res => res.json())
  .then(data => {
    if (!Array.isArray(data)) return;
    data.forEach(m => {
      const colorOrEmoji = (m.color && (m.color in colorToEmoji)) ? m.color : (m.emoji || '🟢');
      makeMarker([m.lat, m.lng], colorOrEmoji, m.name || null).addTo(allLayer);
    });
    updateMarkerCount();
    goToGlobalView();
  })
  .catch(err => {
    console.error('Failed to load markers.json', err);
    updateMarkerCount();
  });
</script>
</body>
</html>

