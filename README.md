// ============================================================
// Karachi Elevation Map - SRTM DEM
// Data: USGS SRTMGL1_003 (30 m)
// Output: elevation map with legend, north arrow and scale bar,
//         elevation statistics, GeoTIFF export
// ============================================================

// 1. Study area
var area = ee.FeatureCollection('users/USERNAME/karachi_boundary');  // boundary asset path
var geom = area.geometry();

// 2. Elevation
var dem = ee.Image('USGS/SRTMGL1_003').select('elevation').clip(geom);
var vis = {
  min: 0,
  max: 500,
  palette: ['08306b', '2171b5', '41b6c4', 'a1dab4', 'ffffb2', 'fecc5c', 'fd8d3c', 'e31a1c']
};

// 3. Map layers
Map.centerObject(area, 10);
Map.addLayer(dem, vis, 'Elevation (m)');
Map.addLayer(area.style({color: '000000', fillColor: '00000000', width: 2}), {}, 'Boundary');

// 4. Legend
function boxStyle(position) {
  return {position: position, padding: '6px 10px', backgroundColor: 'white'};
}

var legend = ui.Panel({style: boxStyle('bottom-left')});
legend.add(ui.Label('Elevation (m)', {fontWeight: 'bold', margin: '0 0 4px 0'}));
legend.add(ui.Thumbnail({
  image: ee.Image.pixelLonLat().select(0),
  params: {bbox: [0, 0, 1, 0.1], dimensions: '200x16', format: 'png',
           min: 0, max: 1, palette: vis.palette},
  style: {margin: '0'}
}));
legend.add(ui.Panel([
  ui.Label(String(vis.min), {width: '75px', margin: '2px 0'}),
  ui.Label(String((vis.min + vis.max) / 2), {width: '50px', margin: '2px 0', textAlign: 'center'}),
  ui.Label(vis.max + '+', {width: '75px', margin: '2px 0', textAlign: 'right'})
], ui.Panel.Layout.flow('horizontal')));
legend.add(ui.Panel([
  ui.Label('', {width: '20px', height: '10px', margin: '4px 6px 0 0', border: '2px solid black'}),
  ui.Label('Boundary', {margin: '0'})
], ui.Panel.Layout.flow('horizontal')));
Map.add(legend);

// 5. North arrow
Map.add(ui.Panel({
  widgets: [
    ui.Label('N', {fontWeight: 'bold', fontSize: '18px', textAlign: 'center',
                   stretch: 'horizontal', margin: '0'}),
    ui.Label('\u25B2', {fontSize: '24px', textAlign: 'center',
                        stretch: 'horizontal', margin: '0'})
  ],
  style: boxStyle('top-right')
}));

// 6. Scale bar (updates on pan/zoom)
var scaleBar = ui.Panel({style: boxStyle('bottom-center')});
Map.add(scaleBar);

function updateScaleBar(view) {
  var mPerPx = 156543.03392 * Math.cos(view.lat * Math.PI / 180) / Math.pow(2, view.zoom);

  // longest round length that fits in about 120 px
  var km = 0.1;
  [0.1, 0.2, 0.5, 1, 2, 5, 10, 20, 50, 100, 200, 500].forEach(function (k) {
    if (k * 1000 <= mPerPx * 120) km = k;
  });
  var half = Math.round(km * 1000 / mPerPx / 2) + 'px';

  function seg(color) {
    return ui.Label('', {width: half, height: '6px', margin: '0',
                         backgroundColor: color, border: '1px solid black'});
  }
  function tick(text, width) {
    var style = {margin: '2px 0 0 0', fontSize: '11px'};
    if (width) style.width = width;
    return ui.Label(text, style);
  }

  scaleBar.clear();
  scaleBar.add(ui.Panel([seg('black'), seg('white')], ui.Panel.Layout.flow('horizontal')));
  scaleBar.add(ui.Panel([tick('0', half), tick(String(km / 2), half), tick(km + ' km')],
                        ui.Panel.Layout.flow('horizontal')));
}
updateScaleBar({lat: 24.9, zoom: 10});
Map.onChangeBounds(updateScaleBar);

// 7. Elevation statistics
var stats = dem.reduceRegion({
  reducer: ee.Reducer.minMax().combine({reducer2: ee.Reducer.mean(), sharedInputs: true}),
  geometry: geom,
  scale: 30,
  maxPixels: 1e13,
  tileScale: 4
});
print('Elevation statistics (m)', stats);

// 8. Export
Export.image.toDrive({
  image: dem,
  description: 'Karachi_Elevation_SRTM',
  folder: 'Karachi_Tree_Research',
  region: geom,
  scale: 30,
  maxPixels: 1e13,
  fileFormat: 'GeoTIFF'
});
