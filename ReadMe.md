# WCS and NULL Values

This example Mapfile tests setting NULL data values for WCS requests using MapServer. 
Sample tiff file from https://files.isric.org/soilgrids/latest/ - thanks to ISRIC – World Soil Information. 

To run the examples below require MapServer and GDAL to be installed. Any 7.x or 8.0 version of MapServer should produce the same output. 

## WCS 1.x

A NULL data value can be set when using WCS 1.x using layer metadata in the Mapfile:

```
METADATA
	"wcs_rangeset_nullvalue" "-32768"
END
```

In the code this sets the `FORMATOPTION "NULLVALUE=-32768"` dynamically on the `OUTPUTFORMAT` requested by the client. 

Note this is for **WCS 1.x only** as mentioned in the [WCS Server docs](https://mapserver.org/ogc/wcs_server.html#layer-object-metadata):

> wcs_rangeset_nullvalue
> Description: (Optional, for DescribeCoverage request) Rangeset single null value (WCS 1.x only)

The `FORMATOPTION "NULLVALUE=-32768"` line can be commented out in `OUTPUTFORMAT` to confirm just setting the layer metadata works correctly. 
Now try running the following commands:

```
# note for MapServer 8.0 the MAPSERVER_CONFIG_FILE environment variable may need to be set
# export MAPSERVER_CONFIG_FILE=mapserver.conf
mapserv -nh "QUERY_STRING=map=test.map&SERVICE=WCS&VERSION=1.0.0&REQUEST=GetCoverage&Coverage=test&FORMAT=GEOTIFF_INT16&BBOX=-69.955,3.420,-69.701,3.5896&CRS=EPSG:4326&WIDTH=500&HEIGHT=500" > output.tif
gdalinfo output.tif
```

The `gdalinfo` output should include the **NoData Value=-32768**:

```	
Driver: GTiff/GeoTIFF
Files: D:\Temp\output.tif
...
Corner Coordinates:
Upper Left  ( -70.8104920,   4.4983138) ( 70d48'37.77"W,  4d29'53.93"N)
Lower Left  ( -70.8104920,   3.1505088) ( 70d48'37.77"W,  3d 9' 1.83"N)
Upper Right ( -69.4118681,   4.4983138) ( 69d24'42.73"W,  4d29'53.93"N)
Lower Right ( -69.4118681,   3.1505088) ( 69d24'42.73"W,  3d 9' 1.83"N)
Center      ( -70.1111801,   3.8244113) ( 70d 6'40.25"W,  3d49'27.88"N)
Band 1 Block=256x256 Type=Int16, ColorInterp=Gray
  NoData Value=-32768
```


## WCS 2.0

### Global OUTPUTFORMAT Setting

The layer metadata approach described above is not implemented for WCS 2.0. One option is to set a NULL data value can be set directly on the `OUTPUTFORMAT` option. See 
the [OUTPUTFORMAT docs](https://mapserver.org/mapfile/outputformat.html):

> GDAL/*: “NULLVALUE=n” is used in raw image modes (IMAGEMODE BYTE/INT16/FLOAT) to pre-initialize the raster and an attempt 
> is made to record this in the resulting file as the nodata value. This is automatically set in WCS mode if rangeset_nullvalue is set.

The note *"this is automatically set in WCS mode if rangeset_nullvalue is set"* only applies for WCS 1.x. 

An example `OUTPUTFORMAT`:

```
OUTPUTFORMAT
	NAME "GEOTIFF_INT16"
	DRIVER "GDAL/GTiff"
	MIMETYPE "image/tiff"
	FORMATOPTION "COMPRESS=DEFLATE"
	FORMATOPTION "ZLEVEL=2"
	FORMATOPTION "TILED=YES"
	FORMATOPTION "PREDICTOR=2"
	FORMATOPTION "NULLVALUE=-32768" # set null value here
	IMAGEMODE INT16
	EXTENSION "tif"
END
```

This approach has the limitation that different layers cannot use different NULL values, as all layers share the same `OUTPUTFORMAT` options. 

### Layer Metadata Approach (Recommended)

WCS 2.0 metadata options were updated [with this pull request](https://github.com/MapServer/MapServer/pull/6456) in MapServer 7.2. 
They are documented [here](https://mapserver.org/ogc/wcs_server.html#layer-object-metadata):

> wcs_outputformat_{OUTPUTFORMATNAME}_creationoption_{OPTIONNAME}
>
> Description: (Optional) (added in MapServer 7.2.0) To provide creation options in a (GDAL) output format 
> specific and layer specific way. {OUTPUTFORMATNAME} should be substituted by the user with the OUTPUTFORMAT.NAME 
> value of an output format allowed for the layer. And {OPTIONNAME} by a valid GDAL creation option valid for the OUTPUTFORMAT.DRIVER.
>
> The purpose of this metadata item is to provide creation options that depend on the layer, typically to configure some metadata in 
> the resulting raster file that depends on that layer (for non-layer dependent configuration options, OUTPUTFORMAT.FORMATOPTION should rather be used).

Example layer metadata:

```	
METADATA
	# this metadata item is in the format wcs_outputformat_{OUTPUTFORMAT}_creationoption_{FORMATOPTION}		
	"wcs_outputformat_GEOTIFF_INT16_creationoption_nullvalue" "-32768" # WCS 2.0 only
END
```

Again in the code this sets the `FORMATOPTION "NULLVALUE=-32768"` dynamically on the `OUTPUTFORMAT` requested by the client. 

Running the following command to check this:

```
mapserv -nh "QUERY_STRING=map=test.map&SERVICE=WCS&VERSION=2.0.0&REQUEST=GetCoverage&CoverageId=test&FORMAT=GEOTIFF_INT16&BBOX=-69.955,3.420,-69.701,3.5896&CRS=EPSG:4326&WIDTH=500&HEIGHT=500" > output2.tif
gdalinfo output.tif
```

Output:

```
Band 1 Block=256x256 Type=Int16, ColorInterp=Gray
  NoData Value=-32768
```

If debugging is set to on the logs should have output confirming the option has been set:

```
[Thu Feb  9 11:58:56 2023].796000 Setting GDAL nullvalue=-32768 creation option
```