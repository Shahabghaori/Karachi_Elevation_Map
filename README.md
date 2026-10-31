// ============================================================
// KARACHI ELEVATION MAP — SRTM DEM
// Data source: USGS SRTM Digital Elevation Model
// Output: Elevation map (with legend, scale bar, north arrow) and GeoTIFF export
// ============================================================

// 1. Load Karachi study area
var studyArea = ee.FeatureCollection(
  'projects/causal-galaxy-482405-n9/assets/out'
);
var boundary = studyArea.geometry();

// 2. Load elevation data
var elevation = ee.Image('USGS/SRTMGL1_003')
  .select('elevation')
  .clip(boundary);

// 3. Display elevation map
var palette = [
  '08306b', '2171b5', '41b6c4', 'a1dab4',
  'ffffb2', 'fecc5c', 'fd8d3c', 'e31a1c'
];
var visParams = {min: 0, max: 500, palette: palette};

Map.centerObject(studyArea, 10);
Map.addLayer(elevation, visParams, 'Karachi Elevation (m)');

// 4. Display study area boundary
Map.addLayer(
  studyArea.style({color: '000000', fillColor: '00000000', width: 2}),
  {},
  'Karachi Boundary'
);

// ============================================================
// 5. MAP FURNITURE: LEGEND, NORTH ARROW, SCALE BAR
// ============================================================

// ---------- 5a. LEGEND (bottom-left) ----------
var legend = ui.Panel({
  style: {
    position: 'bottom-left',
    padding: '8px 12px',
    backgroundColor: 'rgba(255,255,255,0.9)'
  }
});

legend.add(ui.Label({
  value: 'Elevation (m)',
  style: {fontWeight: 'bold', fontSize: '14px', margin: '0 0 6px 0'}
}));

// Gradient colour bar
var colorBar = ui.Thumbnail({
  image: ee.Image.pixelLonLat().select(0),
  params: {
    bbox: [0, 0, 1, 0.1],
    dimensions: '200x16',
    format: 'png',
    min: 0,
    max: 1,
    palette: palette
  },
  style: {stretch: 'horizontal', margin: '0px 0px', maxHeight: '20px'}
});
legend.add(colorBar);

// Min / mid / max labels
legend.add(ui.Panel({
  widgets: [
    ui.Label(String(visParams.min), {margin: '2px 0', fontSize: '12px'}),
    ui.Label(String((visParams.min + visParams.max) / 2), {
      margin: '2px 0', fontSize: '12px', textAlign: 'center', stretch: 'horizontal'
    }),
    ui.Label(visParams.max + '+', {margin: '2px 0', fontSize: '12px'})
  ],
  layout: ui.Panel.Layout.flow('horizontal')
}));

// Boundary entry
legend.add(ui.Panel({
  widgets: [
    ui.Label('', {
      width: '24px', height: '12px', margin: '4px 6px 0 0',
      border: '2px solid black'
    }),
    ui.Label('Karachi boundary', {margin: '2px 0', fontSize: '12px'})
  ],
  layout: ui.Panel.Layout.flow('horizontal')
}));

Map.add(legend);

// ---------- 5b. NORTH ARROW (top-right) ----------
var northArrow = ui.Panel({
  widgets: [
    ui.Label('N', {
      fontWeight: 'bold', fontSize: '20px', textAlign: 'center',
      stretch: 'horizontal', margin: '0', backgroundColor: 'rgba(0,0,0,0)'
    }),
    ui.Label('▲', {
      fontSize: '26px', textAlign: 'center',
      stretch: 'horizontal', margin: '0', backgroundColor: 'rgba(0,0,0,0)'
    })
  ],
  style: {
    position: 'top-right',
    padding: '4px 10px',
    backgroundColor: 'rgba(255,255,255,0.85)'
  }
});
Map.add(northArrow);

// ---------- 5c. DYNAMIC SCALE BAR (bottom-center) ----------
var scalePanel = ui.Panel({
  style: {
    position: 'bottom-center',
    padding: '6px 10px',
    backgroundColor: 'rgba(255,255,255,0.85)'
  }
});
Map.add(scalePanel);

function updateScaleBar(view) {
  // Ground resolution (m per screen pixel) at current latitude and zoom
  var metersPerPx = 156543.03392 * Math.cos(view.lat * Math.PI / 180) /
                    Math.pow(2, view.zoom);

  // Choose a "nice" bar length that is roughly 120 px wide
  var targetMeters = metersPerPx * 120;
  var niceKm = [0.1, 0.2, 0.5, 1, 2, 5, 10, 20, 50, 100, 200, 500];
  var lengthKm = niceKm[0];
  for (var i = 0; i < niceKm.length; i++) {
    if (niceKm[i] * 1000 <= targetMeters) lengthKm = niceKm[i];
  }

  var totalPx = Math.round(lengthKm * 1000 / metersPerPx);
  var halfPx = Math.round(totalPx / 2);

  scalePanel.clear();

  // Alternating black / white bar
  scalePanel.add(ui.Panel({
    widgets: [
      ui.Label('', {
        width: halfPx + 'px', height: '6px', margin: '0',
        backgroundColor: 'black', border: '1px solid black'
      }),
      ui.Label('', {
        width: halfPx + 'px', height: '6px', margin: '0',
        backgroundColor: 'white', border: '1px solid black'
      })
    ],
    layout: ui.Panel.Layout.flow('horizontal')
  }));

  // Tick labels: 0, half, full
  scalePanel.add(ui.Panel({
    widgets: [
      ui.Label('0', {width: halfPx + 'px', margin: '2px 0 0 0', fontSize: '11px',
                     backgroundColor: 'rgba(0,0,0,0)'}),
      ui.Label(String(lengthKm / 2), {width: halfPx + 'px', margin: '2px 0 0 0',
                     fontSize: '11px', backgroundColor: 'rgba(0,0,0,0)'}),
      ui.Label(lengthKm + ' km', {margin: '2px 0 0 0', fontSize: '11px',
                     backgroundColor: 'rgba(0,0,0,0)'})
    ],
    layout: ui.Panel.Layout.flow('horizontal')
  }));
}

// Draw once, then redraw whenever the map is panned or zoomed
updateScaleBar({lon: 67.0, lat: 24.9, zoom: 10});
Map.onChangeBounds(updateScaleBar);

// ============================================================
// 6. Print elevation statistics
// ============================================================
var elevationStats = elevation.reduceRegion({
  reducer: ee.Reducer.minMax()
    .combine({reducer2: ee.Reducer.mean(), sharedInputs: true}),
  geometry: boundary,
  scale: 30,
  maxPixels: 1e13,
  tileScale: 4
});
print('Karachi Elevation Statistics (metres)', elevationStats);

// 7. Export elevation map to Google Drive
Export.image.toDrive({
  image: elevation,
  description: 'Karachi_Elevation_SRTM',
  folder: 'Karachi_Tree_Research',
  fileNamePrefix: 'Karachi_Elevation_SRTM',
  region: boundary,
  scale: 30,
  maxPixels: 1e13,
  fileFormat: 'GeoTIFF'
});

// ============================================================
// END
// ============================================================
