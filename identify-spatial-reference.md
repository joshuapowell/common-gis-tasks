# What to do when you encounter coordinates that you don't recognize?

## Overview
When dealing with open data you will often be faced with coordinates that you don't recognize. 
This document provides basic sluething tactics for identifying and converting coordinate formats
from unknown types into a web-compatible format.

## Purpose
To identify a previously unknown coordinate format (spatial reference) and convert it into a
[WGS-84 (4326)](https://epsg.io/4326).

## Identify Original Spatial Reference
It is paramont that you are able to identify the original spatial reference in order to convert
a file to your desired format.

### Shapefile
To identify the spatial reference of a Shapefile, you will need to open the package/directory and
locate the projection file this will be the file with the `.prj` extension.

You should be able to open this `.prj` file with any plain text editor or code editor (i.e., VI,
Sublime, Atom, TextMate, TextWrangler).

My example shapefile looks like this:

```
PROJCS[
  "NAD_1983_StatePlane_Pennsylvania_South_FIPS_3702_Feet",
  GEOGCS[
    "GCS_North_American_1983",
    DATUM[
      "D_North_American_1983",
      SPHEROID["GRS_1980",6378137.0,298.257222101]
    ],
    PRIMEM["Greenwich",0.0],
    UNIT["Degree",0.0174532925199433]
  ],
  PROJECTION["Lambert_Conformal_Conic"],
  PARAMETER["False_Easting",1968500.0],
  PARAMETER["False_Northing",0.0],
  PARAMETER["Central_Meridian",-77.75],
  PARAMETER["Standard_Parallel_1",39.93333333333333],
  PARAMETER["Standard_Parallel_2",40.96666666666667],
  PARAMETER["Latitude_Of_Origin",39.33333333333334],
  UNIT["Foot_US",0.3048006096012192]
]
```

The second line, in our case `"NAD_1983_StatePlane_Pennsylvania_South_FIPS_3702_Feet"` gives us the projection of our shapefile.

Our goal is to take this information and identify the EPSG numeric code or "Spatial Reference ID".

### Searching for your Spatial Reference ID
1. Using your browser visit http://spatialreference.org/
2. This website provides a search in the top right, we will start with broad searches by entering "NAD 1983" as our first search‡
3. Our second search will be "NAD 1983 Pennsylania South"
4. By this time you should have only a few results, `EPSG:2272` should be the top result on this website.
5. `2272` is the spatial reference we are looking for.

‡ *Note: If you copy and paste the above line `"NAD_1983_StatePlane_Pennsylvania_South_FIPS_3702_Feet"` you will receive a result that is not EPSG but rather SR-ORG. SR-ORG will not work for our purposes, remember our goal is to identify the EPSG spatial reference id.*
  PARAMETER["False_Easting",1968500.0],
  
### How can we utilize our newly discoverd spatial reference?
If you import your shapefile into a PostgreSQL database with PostGIS installed, you can convert the file's coordinates.

To convert and import the shapefile into the database execute these commands, making sure to replace our placeholder file and database names with your own.

Convert the Shapefile to SQL
```
shp2pgsql -s 4326 ~/MY_SHAPEFILE_NAME.shp TABLE_NAME postgres > TABLE_NAME.sql
```

Execute the SQL Commands
```
psql -d DATABASE_NAME -U postgres -p 5432 -f TABLE_NAME.sql
```

#### Exporting Your Values as GeoJSON
To view your data in WGS-84 format without actually changing the original dataset, you can execute the following command. Remember that in this example we are using the Spatial Reference `2272` which we identified in the steps above, your spatial reference may be different.

```
SELECT ST_AsGeoJSON(ST_AsEWKT(ST_Transform(ST_GeomFromEWKT(CONCAT('SRID=2272;', ST_AsText(TABLE_NAME.geom))),4326))) FROM TABLE_NAME;
```

Executing this command from *our* dataset produces the following response.

```
{
  "type":"MultiLineString",
  "coordinates":[[[-79.9212979256631,40.4628710326199],[-79.9204726983908,40.464094002471],[-79.9198562674797,40.4651311357558],[-79.9192458274249,40.4661296164225],[-79.9188008010287,40.4662952587881],[-79.9178832579189,40.4666344153159],[-79.9169744513377,40.4668821029018],[-79.9161736102166,40.4670495034741],[-79.9151203614502,40.4672497707719],[-79.9140661466792,40.4676549123141],[-79.9126885978815,40.4682425913432],[-79.9115492612404,40.4687338564085],[-79.9107487070489,40.4692065875645],[-79.9099378882867,40.4698872694003],[-79.9095444190316,40.4702028367597],[-79.909234667301,40.4702978607643],[-79.9086178006764,40.4703457617809]]]
}
```

To export your entire table as a ready-to-use GeoJSON Feature Collection, you can execute.

Remember to replace `TABLE_NAME` with your PostgreSQL database table name and `COLUMN_NAME_1` with your desired columns. Lastly, remember to change `2272` to your identified spatial reference.

```
SELECT 
  row_to_json(fc)
FROM (
  SELECT 
    'FeatureCollection' AS type, 
    array_to_json(array_agg(f)) AS features
  FROM (
    SELECT 
      'Feature' AS type, 
      ST_AsGeoJSON(ST_AsEWKT(ST_Transform(ST_GeomFromEWKT(CONCAT('SRID=2272;', ST_AsText(lg.geom))),4326)))::json AS geometry,
      row_to_json((SELECT l FROM (SELECT COLUMN_NAME_1, COLUMN_NAME_2, COLUMN_NAME_3) As l)) AS properties
    FROM 
      TABLE_NAME AS lg
  ) AS f
)  AS fc;
```
