---
layout: post
title: "Waiting for Postgis 3.1: Performance with large geometries"
tags: [postgis, performance]
comments: false
---

Today I want to talk about partial decompression in PostGIS and how it has improved the speed of some operations with large geometries, that is the geometries made of hundreds of points or vertices. I don't think we've intentionally focused on performance for the upcoming PostGIS 3.1 but unconsciously I certainly did so I get to talk about it.

Previous posts in this series:
* [Postgis 3.1: Improvements on MVT functions](https://rmr.ninja/2020-11-19-waiting-for-postgis-3-1-mvt/)
* [Postgis 3.1: Performance Improvements on output functions](https://rmr.ninja/2020-12-06-waiting-for-postgis-3-1-output/)

## Disk storage

When PostGIS needs to store geospatial data in the database it does so by [serializing it](https://en.wikipedia.org/wiki/Serialization) into a PostgreSQL [varlena](https://doxygen.postgresql.org/structvarlena.html), which is just a variable length array.

There are currently two serialization methods in PostGIS but since v2 is an extension of v1 I'll just give a quick overview of the structure the aforementioned array, which looks like this:

| Size | SRID | Flags | ExtendedFlags | BoundingBox | GeometryData |


The `ExtendedFlags` and `BoundingBox` are optional (enabled in the first `Flags`), and the format and size of `GeometryData` will vary depending on the geometry type, number of subgeometries, rings, etc. What's important is that at the end of the process we have a continuous array with a known size that we can feed to the database to handle however it sees fit.

And in this "however it sees fit" is the crux of the matter as PostgreSQL does it in a couple of different ways: For small values it will store the geometry next to the other columns in the row, but if the array reaches a certain size (normally 2 kB) it will be [TOASTed](https://www.postgresql.org/docs/13/storage-toast.html), which for PostGIS types means they will be compressed and only stored with other columns if they are small enough, otherwise they will be moved to out of line storage.

This is defined during the declaration of the types themselves:

```sql
CREATE TYPE geometry (
	internallength = variable,
	input = geometry_in,
	output = geometry_out,
	send = geometry_send,
	receive = geometry_recv,
	typmod_in = geometry_typmod_in,
	typmod_out = geometry_typmod_out,
	delimiter = ':',
	alignment = double,
	analyze = geometry_analyze,
	storage = main
);
```

The `geometry` type is created as a `internallength = variable`, that is with variable length, and with `storage = main` so when the TOAST size limit is reached, the `main` [strategy](https://www.postgresql.org/docs/13/storage-toast.html#68.2.1) will be used. So in order to work with a geometry that has been toasted we need to first read it from disk, then decompress it and finally we need to convert the serialized array into the on-memory format.

As a way to speed up some operations, when the information that PostGIS needs is available in the headers (everything but the `GeometryData`) we avoid the deserialization process and read whatever bytes necessary directly from the serialized array. This improves the performance of actions such as knowing the type or the bounding box of a geometry, but it has a much smaller impact on toasted values. In those cases, we might spend 10 ms reading and decompressing information from the disk just so we can do a really fast operation over a tiny fraction of that.

In the past there have been prototypes using [different storage strategies](http://blog.cleverelephant.ca/2018/09/postgis-external-storage.html) but it's hard to strike a balance between how much disk space is used, the disk speed and the CPU speed decompressing it.in all cases. As this situation affects any variable length types in PostgreSQL , in recent years there proposals such as allowing [custom compression methods](https://www.postgresql.org/message-id/flat/20170907194236.4cefce96%40wp.localdomain) or improving [pglz performance](https://www.postgresql.org/message-id/flat/469C9ED9-348C-4FE7-A7A7-B0FA671BEE4C%40yandex-team.ru).

## PG 12 changes

In the same blog post that I referenced in regards of [testing different strategies](http://blog.cleverelephant.ca/2018/09/postgis-external-storage.html) Paul Ramsey, co-founder of the PostGIS project, mentioned that a possible solution would be to avoid doing work completely and stop decompressing once you have the bytes hat you are interested in, which in our case is the header size. Being the madlad that he is, he [proposed a patch](https://www.postgresql.org/message-id/flat/CACowWR07EDm7Y4m2kbhN_jnys%3DBBf9A6768RyQdKm_%3DNpkcaWg%40mail.gmail.com) to add partial decompression support to PostgreSQL and it landed in PG12. Awesome.

## PostGIS 3.1 changes

Although the patch landed in PostgreSQL 12 there was still work that needed to be done to achieve the full potential of the improvement. Before, you see, there wasn't a good reason to request only the header since that wasn't any faster than requesting it whole, in fact getting just the header was slower as it required an extra copy of the data. Therefore, once the patch was applied, we had to locate which processes would benefit from partial decompression and change them, which I did in [this PR](https://github.com/postgis/postgis/pull/558) affecting the following functions calls:

* [Bounding box operators](https://postgis.net/docs/reference.html#idm9874) like [&&](https://postgis.net/docs/geometry_overlaps.html). Note that only the non index scan cases are improved as indices store the bounding box separately and don't need to read it from the data itself.

* Functions that only need to read the headers: [`ST_SRID`](https://postgis.net/docs/ST_SRID.html),  [`ST_NDims`](https://postgis.net/docs/ST_NDims.html), [`PostGIS_HasBBox`](https://postgis.net/docs/PostGIS_HasBBox.html), `SE_Is3D` or `SE_IsMeasured`.

* As part of the PR but unrelated to decompression, I also improved the speed to cast a [PostGIS point into a PostgreSQL point](https://github.com/postgis/postgis/blob/a7053ec1333935d0afcf78d7f56eb197b8ff3db5/postgis/postgis.sql.in#L180) by operating over serialized data directly.

This was a manual work guided by experience so it's likely that other functions could also benefit from similar patches in the future. I know for a fact that functions like [`ST_NPoints`](https://postgis.net/docs/ST_NPoints.html) or [`ST_NRings`](https://postgis.net/docs/ST_NRings.html) could be rewritten to work only with the geometry header.

As a side note, a few months after integrating the change I detected a performance regression over small geometries introduced by this change. Whereas the old function returned the buffer itself when it didn't need to process it, the partial decompression method returns a copy of the buffer incurring a performance penalty. The fix ([#586](https://github.com/postgis/postgis/pull/586)) was as simple as only calling the decompression method when the datum is compressed.

## Performance comparison

To measure the impact of the changes, I'll again be using the [Boundaries of Canada Provinces](https://open.canada.ca/data/en/dataset/a883eb14-0c0e-45c4-b8c4-b54c4a819edb) dataset:

```bash
$ shp2pgsql -D -s 4326 -I lpr_000b16a_e/lpr_000b16a_e canada | psql -U postgres benchmarks
```

All the rows of the dataset (13) are toasted since they are bigger than the 2 kB threshold:

```sql
SELECT
    pg_size_pretty(avg(pg_column_size(geom)::bigint)) as avg_size,
    pg_size_pretty(min(pg_column_size(geom)::bigint)) as min_size,
    pg_size_pretty(max(pg_column_size(geom)::bigint)) as max_size
FROM canada;
 avg_size | min_size | max_size 
----------+----------+----------
 3015 kB  | 74 kB    | 15 MB
(1 row)
```


### Queries

To run the tests I'm using pgbench and running one query over and over for 30 seconds and comparing the query latency between PostGIS 3.0 and 3.1.

```sql
$ cat large.pgbench
EXPLAIN (ANALYZE, TIMING OFF) SELECT ST_SRID(geom) FROM canada; -- ST_SRID
--EXPLAIN (ANALYZE, TIMING OFF) SELECT ST_NDims(geom) FROM canada; -- ST_NDims
--EXPLAIN (ANALYZE, TIMING OFF) SELECT * FROM canada WHERE geom && ST_TileEnvelope(3,1,1); -- Tiling
--EXPLAIN (ANALYZE, TIMING OFF) SELECT a.gid, b.gid, a.geom && a.geom FROM canada a, canada b; -- geom && geom
$ pgbench -c 1 -T 30 -r -f large.pgbench  -U postgres benchmarks
...
statement latencies in milliseconds:
         9.247  EXPLAIN (ANALYZE, TIMING OFF) SELECT ST_SRID(geom) FROM canada;
```

### Results

| Query | 3.0 latency (ms) | 3.1 latency (ms) | Change |
| :------ |:--- | :--- | :--- |
| [`ST_SRID`](https://postgis.net/docs/ST_SRID.html) | 116.064 | 9.247 | **12.552x** |
| [`ST_NDims`](https://postgis.net/docs/ST_NDims.html) | 115.903 | 8.907 | **13.013x** |
| Tiling | 119.533 | 8.906 | **13.421x** |
| geom `&&` geom | 3189.462 | 189.105 | **16.866x** |

Note that in these cases the improvement will be bigger the faster the disk is as the extra decompression would have taken a bigger slice of the time. For example, when I initially tested the path I was seeing a **120 to 470x** improvement, but I was using a PC with better hardware back then and currently I'm using a more limited laptop. Or at least I hope that's the reason for the big discrepancies in both the 3.0 and 3.1 runs between the benchmarks a few months ago and the benchmarks today.

Although this is "just" a corner case, it's nice to see PostGIS working better with large geometries. After all, a round world is cornerless.
