# spatial_analysis_challenge

2023-08-25
Spatial Analysis challenge
https://twitter.com/spatialthoughts/status/1695021747177951435

By Bert Temme
Using GeoParquet and DuckDB

## Create GeoParquet files
```
$ duckdb 
D load spatial;
D create table schools as select * from st_read('data.gpkg', layer='schools');
D create table admin_boundaries as select * from st_read('data.gpkg', layer='admin_boundaries');
D create table colleges as select * from st_read('data.gpkg', layer='colleges');
```
## spatial join the schools and admin areas

```
D create table schools1 as SELECT s.osm_id as school_id, ST_GeomFromWKB(s.geom) AS school_geom, a.shapename as shapename
FROM schools s
JOIN admin_boundaries a ON ST_Intersects(ST_GeomFromWKB(s.geom), ST_GeomFromWKB(a.geom));
```

## spatial join the colleges and admin areas

```
D create table colleges1 as SELECT c.osm_id as college_id, ST_GeomFromWKB(c.geom) AS college_geom, a.shapename as shapename
FROM colleges c
JOIN admin_boundaries a ON ST_Intersects(ST_GeomFromWKB(c.geom), ST_GeomFromWKB(a.geom));
```

## join colleges and schools on admin_boundary name

```
D create table admin_schools_colleges as select s.shapename, c.shapename,  s.school_id as school_id, c.college_id as college_id, st_distance(c.college_geom, s.school_geom) as distance 
from schools1 s 
left join colleges1 c on 
c.shapename=s.shapename; 
```

## select the schools and nearest colleges

```
D create table schools_colleges as 
SELECT  a.school_id as school_id, a.college_id as college_id
FROM  admin_schools_colleges  a
INNER JOIN
(
    select a.school_id as school_id, min(a.distance) as distance
    from admin_schools_colleges a
    group by a.school_id
) b ON a.school_id = b.school_id AND a.distance = b.distance;
```

## create result table, make the lines

exclude the schools without a college in the admin_boundary

```
D create table result as select st_aswkb(st_makeline(ST_GeomFromWKB(s.geom),  ST_GeomFromWKB(c.geom))) as geometry
from schools_colleges a
left join schools s on a.school_id=s.osm_id
left join colleges c on a.college_id=c.osm_id
where c.geom is not null;
```

## export to parquet

```
D COPY result TO 'result.parquet' (FORMAT PARQUET);
```

## Convert to GeoParquet

![image](https://github.com/bertt/spatial_analysis_challenge/assets/538812/2685612d-c48c-43f9-83dd-4dd386d7478c)


```
$ gpq convert result.parquet result1.parquet
```

## Load GeoParquet into QGIS
