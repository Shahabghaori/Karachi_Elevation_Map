// ============================================================
// KARACHI ELEVATION MAP — SRTM DEM
// Data source: USGS SRTM Digital Elevation Model
// Output: Elevation map and GeoTIFF export
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
Map.centerObject(studyArea, 10);

Map.addLayer(
  elevation,
  {
    min: 0,
    max: 500,
    palette: [
      '08306b',
      '2171b5',
      '41b6c4',
      'a1dab4',
      'ffffb2',
      'fecc5c',
      'fd8d3c',
      'e31a1c'
    ]
  },
  'Karachi Elevation (m)'
);

// 4. Display study area boundary
Map.addLayer(
  studyArea.style({
    color: '000000',
    fillColor: '00000000',
    width: 2
  }),
  {},
  'Karachi Boundary'
);

// 5. Print elevation statistics
var elevationStats = elevation.reduceRegion({
  reducer: ee.Reducer.minMax()
    .combine({
      reducer2: ee.Reducer.mean(),
      sharedInputs: true
    }),
  geometry: boundary,
  scale: 30,
  maxPixels: 1e13,
  tileScale: 4
});

print('Karachi Elevation Statistics (metres)', elevationStats);

// 6. Export elevation map to Google Drive
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
