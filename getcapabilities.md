# WCS GetCapabilities

Returning GetCapabilities for a WCS service can be very slow, for example when calling: https://maps.isric.org/mapserv?map=/map/nitrogen.map&request=getcapabilities&service=wcs

To get GetCapabilities on the command line with the `test.map` used in this project:

```
# note for MapServer 8.0 the MAPSERVER_CONFIG_FILE environment variable may need to be set
cd D:\GitHub\wcs-test
set MAPSERVER_CONFIG_FILE=mapserver.conf
mapserv -nh "QUERY_STRING=map=test.map&SERVICE=WCS&VERSION=2.0.0&REQUEST=getcapabilities"
```

Debugging this call highlighted the issue. [msWCSGetCoverageMetadata20](https://github.com/MapServer/MapServer/blob/dfd24a99affc1279f380df78ba206f965aab0d01/mapwcs20.cpp#L2536) is called for each layer.
If metadata values are not set then GDAL is used to open each raster layer in the Mapfile to calculate the metadata properties.

This can be skipped by ensuring the `extent` and either `resolution` or `size` are set in the metadata properties of the layer. 

```c
  if( msOWSLookupMetadata(&(layer->metadata), "CO", "extent") != NULL
      && (msOWSLookupMetadata(&(layer->metadata), "CO", "resolution") != NULL
          || msOWSLookupMetadata(&(layer->metadata), "CO", "size") != NULL) ) {
```

An example of setting the metadata items is given below:

```
LAYER
    NAME "test"
    METADATA
        # require the following to be set for each layer
        "ows_extent" "-19949000.0 -6147500.0 19861750.0 8361000.0"
        # only one of the following is required to avoid reading the raster to calculate metadata
        "ows_resolution" "0.002209516370912 -0.002209516370912" # Pixel Size from gdalinfo
        "ows_size" "633 610" # or wcs_size from gdalinfo - Size is 633, 610
```

As per the [MapServer WCS docs](https://www.mapserver.org/ogc/wcs_server.html#specifying-coverage-specific-metadata):

    The convention is that once (wcs|ows)_extent and one of (wcs|ows)_size and (wcs|ows)_resolution is set in the layer metadata, all the coverage specific metadata will be retrieved from there. Otherwise the source image is queried via GDAL, if possible.

    The relevant layer metadata fields are (wcs|ows)_bandcount, (wcs|ows)_imagemode, (wcs|ows)_native_format, and all New band related metadata entries.

The raster metadata items can be calculated using GDAL - `gdalinfo test.tif`:

```bat
Driver: GTiff/GeoTIFF
Files: D:\GitHub\wcs-test\test.tif
Size is 633, 610
Coordinate System is:
GEOGCRS["WGS 84",
    DATUM["World Geodetic System 1984",
        ELLIPSOID["WGS 84",6378137,298.257223563,
            LENGTHUNIT["metre",1]]],
    PRIMEM["Greenwich",0,
        ANGLEUNIT["degree",0.0174532925199433]],
    CS[ellipsoidal,2],
        AXIS["geodetic latitude (Lat)",north,
            ORDER[1],
            ANGLEUNIT["degree",0.0174532925199433]],
        AXIS["geodetic longitude (Lon)",east,
            ORDER[2],
            ANGLEUNIT["degree",0.0174532925199433]],
    ID["EPSG",4326]]
Data axis to CRS axis mapping: 2,1
Origin = (-70.810491990260900,4.498313785228503)
Pixel Size = (0.002209516370912,-0.002209516370912)
Metadata:
  AREA_OR_POINT=Area
Image Structure Metadata:
  INTERLEAVE=BAND
Corner Coordinates:
Upper Left  ( -70.8104920,   4.4983138) ( 70d48'37.77"W,  4d29'53.93"N)
Lower Left  ( -70.8104920,   3.1505088) ( 70d48'37.77"W,  3d 9' 1.83"N)
Upper Right ( -69.4118681,   4.4983138) ( 69d24'42.73"W,  4d29'53.93"N)
Lower Right ( -69.4118681,   3.1505088) ( 69d24'42.73"W,  3d 9' 1.83"N)
Center      ( -70.1111801,   3.8244113) ( 70d 6'40.25"W,  3d49'27.88"N)
Band 1 Block=633x6 Type=Int16, ColorInterp=Gray
  NoData Value=-32768
```

Alternatively a MapServer `DescribeCoverage` request can be used e.g.

```
mapserv -nh "QUERY_STRING=map=test.map&SERVICE=WCS&VERSION=2.0.1&REQUEST=DescribeCoverage&COVERAGEID=test"
```

Example on a live site: https://maps.isric.org/mapserv?map=/map/nitrogen.map&SERVICE=WCS&VERSION=2.0.1&REQUEST=DescribeCoverage&COVERAGEID=nitrogen_15-30cm_Q0.5

