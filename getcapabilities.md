# WCS GetCapabilities Requests

Returning GetCapabilities for a WCS service can be very slow, for example when calling: https://maps.isric.org/mapserv?map=/map/nitrogen.map&request=getcapabilities&service=wcs

To get GetCapabilities on the command line with the `test.map` used in this project:

```
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

> The convention is that once (wcs|ows)_extent and one of (wcs|ows)_size and (wcs|ows)_resolution is set in the layer metadata, all the coverage specific metadata will be retrieved from there. Otherwise the source image is queried via GDAL, if possible.

> The relevant layer metadata fields are (wcs|ows)_bandcount, (wcs|ows)_imagemode, (wcs|ows)_native_format, and all New band related metadata entries.

Interestingly the XML response doesn't include the size or resolution metadata calculated for each layer. See the
[example response](wcs_capabilities.xml):

```xml
  <wcs:Contents>
    <wcs:CoverageSummary>
      <wcs:CoverageId>test</wcs:CoverageId>
      <wcs:CoverageSubtype>RectifiedGridCoverage</wcs:CoverageSubtype>
    </wcs:CoverageSummary>
  </wcs:Contents>
```

The `msWCSGetCoverageMetadata20` function is reused in a few places, so it was probably simplest to call this each time,
however there is definitely room for optimisation here if only the titles of each layer are required.

The raster metadata items can be calculated using GDAL - `gdalinfo test.tif`:

```shell
Driver: GTiff/GeoTIFF
Files: test.tif
Size is 633, 610
Coordinate System is:
GEOGCRS["WGS 84",
    DATUM["World Geodetic System 1984",
...
Data axis to CRS axis mapping: 2,1
Origin = (-70.810491990260900,4.498313785228503)
Pixel Size = (0.002209516370912,-0.002209516370912)
...
```

Alternatively a MapServer `DescribeCoverage` request can be used to get these details e.g.

```
mapserv -nh "QUERY_STRING=map=test.map&SERVICE=WCS&VERSION=2.0.1&REQUEST=DescribeCoverage&COVERAGEID=test"
```

The `size` parameter is included in the following block (resolution does not appear to be returned):

```xml
<gml:GridEnvelope>
    <gml:low>0 0</gml:low>
    <gml:high>632 609</gml:high>
</gml:GridEnvelope>
```

Example on a live site: https://maps.isric.org/mapserv?map=/map/nitrogen.map&SERVICE=WCS&VERSION=2.0.1&REQUEST=DescribeCoverage&COVERAGEID=nitrogen_15-30cm_Q0.5

