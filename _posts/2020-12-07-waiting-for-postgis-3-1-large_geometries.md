---
layout: post
title: "Waiting for Postgis 3.1: Performance with large geometries"
subtitle: 'When geometries get too big, it's time to call the doctor'
tags: [postgis, performance]
comments: false
---

Previous posts


Serialization -> toasted if size > X. To access the header you needed to read everything from disk, even if it was a simple operation like doing a bbox test. When using indexes the bbox is stored in the index so this isn't needed, but on any other opeartions (like a sequential scan, parallel or no) you had to do the whole process.


## PG 12 changes

What are toasted values and so on. PG12 partial decompression.

## PostGIS 3.1 changes


What were the changes, where and how

## Performance comparison

I'll be again using the [Boundaries of Canada Provinces](https://open.canada.ca/data/en/dataset/a883eb14-0c0e-45c4-b8c4-b54c4a819edb) dataset. Explain size in disk (per geometry => toasted => needs decompression)


```bash
$ shp2pgsql -D -s 4326 -I lpr_000b16a_e/lpr_000b16a_e canada | psql -U postgres benchmarks
```

### Queries

To run the tests I'm using pgbench and running one query over and over for 30 seconds and comparing the query latency between PostGIS 3.0 and 3.1.

```
$ cat large_geoms.pgbench 
XXXXXXXXXXX
XXXXXXXXXXX
XXXXXXXXXXX
XXXXXXXXXXX
XXXXXXXXXXX
$ pgbench -c 1 -T 30 -r -f large_geoms.pgbench  -U postgres benchmarks
...
statement latencies in milliseconds:
       XXXXXXXXXXXXXXXXXXXXXXXXX
```

### Results

| Query | 3.0 latency (ms) | 3.1 latency (ms) | Change |
| :------ |:--- | :--- | :--- |
[ST_AsText](https://postgis.net/docs/ST_AsText.html) default (polygons) | 4962.685 | 551.860 | **9x** |
[ST_AsText](https://postgis.net/docs/ST_AsText.html) default (points) | 590.086 | 222.566 | **2.65x** |
[ST_AsText](https://postgis.net/docs/ST_AsText.html) precision=0 (polygons) | 3192.996 |  556.264 | **5.74x** |
[ST_AsText](https://postgis.net/docs/ST_AsText.html) precision=20 (polygons) | 5914.018 | 552.952 | **10.7x** |
[ST_AsEWKT](https://postgis.net/docs/ST_AsEWKT.html) default (polygons) | 4954.474 | 555.070  | **8.93x** |
[ST_AsGeoJSON](https://postgis.net/docs/ST_AsGeoJSON.html) default (polygons) | 4068.112 | 496.636 | **8.19x** |
[ST_AsGeoJSON](https://postgis.net/docs/ST_AsGeoJSON.html) short CRS (poly) | 4043.459 | 500.971 | **8.07x** |
[ST_AsGeoJSON](https://postgis.net/docs/ST_AsGeoJSON.html) short CRS (points) | 10920.411 | 279.894 | **39.02x** |
[ST_AsGML](https://postgis.net/docs/ST_AsGML.html) v3 default (polygons) | 5267.115 | 893.432 | **5.9x** |
[ST_AsGML](https://postgis.net/docs/ST_AsGML.html) v3 default (points) | 11073.952 | 322.920 | **34.29x** |
[ST_AsSVG](https://postgis.net/docs/ST_AsSVG.html) default (polygons) | 5135.232 | 924.468 | **5.55x** |
[ST_AsSVG](https://postgis.net/docs/ST_AsSVG.html) default (points) | 576.794 | 242.407 | **2.38x** |
[ST_AsX3D](https://postgis.net/docs/ST_AsX3D.html) default (polygons) | 5409.824 | 1157.401 | **4.67x** |
[ST_AsX3D](https://postgis.net/docs/ST_AsX3D.html) default (points) | 10580.443 | 242.407 | **43.65x** |
