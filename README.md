# spatial_analysis_challenge

2023-08-25
Spatial Analysis challenge
https://twitter.com/spatialthoughts/status/1695021747177951435

Connect each school to the nearest college in the same administrative region. 

By Bert Temme
Using GeoParquet and DuckDB

Result live map see https://felt.com/map/Spatial-Analysis-schools-and-colleges-GmBfHzyHR0KZJu9Cge0kbCB

## spatial join the schools and admin areas

```
D create table schools as SELECT s.osm_id as school_id, ST_GeomFromWKB(s.geom) AS school_geom, a.shapename as shapename
FROM st_read('data.gpkg', layer='schools') s
JOIN st_read('data.gpkg', layer='admin_boundaries') a ON ST_Intersects(ST_GeomFromWKB(s.geom), ST_GeomFromWKB(a.geom));
```

## spatial join the colleges and admin areas

```
D create table colleges as SELECT c.osm_id as college_id, ST_GeomFromWKB(c.geom) AS college_geom, a.shapename as shapename
FROM st_read('data.gpkg', layer='colleges') c
JOIN st_read('data.gpkg', layer='admin_boundaries') a ON ST_Intersects(ST_GeomFromWKB(c.geom), ST_GeomFromWKB(a.geom));
```

## join colleges and schools on admin_boundary name

```
D create table admin_schools_colleges as select s.shapename, c.shapename,  s.school_id as school_id, c.college_id as college_id, st_distance(c.college_geom, s.school_geom) as distance 
from schools s 
left join colleges c on 
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
D create table result as select st_aswkb(st_makeline(s.school_geom,  c.college_geom)) as geometry
from schools_colleges a
left join schools s on a.school_id=s.school_id
left join colleges c on a.college_id=c.college_id
where c.college_geom is not null;
```

## export to parquet

```
D COPY result TO 'result.parquet' (FORMAT PARQUET);
```

## Convert to GeoParquet

```
$ gpq convert result.parquet result1.parquet
```

## Load GeoParquet into QGIS

![image](https://github.com/bertt/spatial_analysis_challenge/assets/538812/2685612d-c48c-43f9-83dd-4dd386d7478c)
