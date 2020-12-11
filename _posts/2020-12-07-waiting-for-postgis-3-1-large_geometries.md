---
layout: post
title: "Waiting for Postgis 3.1: Performance with large geometries"
subtitle: 'When geometries get too big, it is time to call the doctor'
tags: [postgis, performance]
comments: false
---

I wouldn't say we have consciuously focused on performance improvements for the upcoming PostGIS 3.1, but I kind of did so I get to talk about it. Previous posts in this series:
* [Postgis 3.1: Improvements on MVT functions](https://rmr.ninja/2020-11-19-waiting-for-postgis-3-1-mvt/)
* [Postgis 3.1: Performance Improvements on output functions](https://rmr.ninja/2020-12-06-waiting-for-postgis-3-1-output/)

Today I want to talk about partial decompression in PostGIS and how it improves the speed of some operations when dealing with large geometries, but to do that I first need to explain a bit how geospatial data is stored in disk and the different mechanisms in place to do it efficiently.

## PostGIS storage

When PostGIS needs to store geospatial data, and currently also when passing it between functions, it does so by [serializing it](https://en.wikipedia.org/wiki/Serialization) into something PostgreSQL knows and can deal with: a variable length array.

There are 2 serialization methods in PostGIS but v2 is an evolution of v1, so I'll just give a quick overview of the most recent:

| Size | SRID | Flags | ExtendedFlags | BoundingBox | GeometryData |


The `ExtendedFlags` and `BoundingBox` are optional (marked in the first flag field), and the `GeometryData` will vary depending on the geometry type, number of subgeometries, rings, etc. What's important is that at the end of the process we have a continuous array of data, with a known size, that we can feed PostgreSQL to store however it sees fit.

And in this "whatever PostgreSQL sees fit" is the crux of the matter as it does it in a couple of different ways: For small values it will store the geometry in the same page as the rest of the columns in the row, but once it grows up to a certain size (normally 2 kB) it will be [TOASTed](https://www.postgresql.org/docs/13/storage-toast.html), which for PostGIS means the geometry will be compressed, and it id doesn't fit in a page it will be moved to out of line storage.

You can see this information in the type creation:

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

The `geometry` type is created as a `internallength = variable`, that is variable length, and with `storage = main` so when the TOAST size limit is reached, the `main` [strategy](https://www.postgresql.org/docs/13/storage-toast.html#68.2.1) will be used. This means that when PostGIS wants to work with a toasted geometry first PostgreSQL needs to read it from disk, then decompress it and finally the buffer needs to be deserialize into the on-memory format.

In order to speed up many operations, when an operation doesn't require the whole geometry, PostGIS can and does work directly with the serialized array, reading the necessary bytes in the header to do common operations such as extracting the type or srid of the geometry, or reading the bounding box to shortcut operations (if the boxes of 2 geometries doesn't intersect the geometries themselves don't intersect and that kind of stuff). The problem comes when to do that simple header check, the data has traveled first from disk (unavoidable, kind of) and fully decompressed as that's a pretty slow operation, so we might have spent 10 ms to be able to do an operation in a thousandth of that.

In the past there have been experiments to use to [different strategies](http://blog.cleverelephant.ca/2018/09/postgis-external-storage.html) but it has drawback: if you don't compress the data you take more space, which can be slower depending on whether reading more from disk is faster or not than decompressing the data. There are other suggestions still going on in PG HACKERS (link here) mailing list to enable custom decompression mechanism (instead of lz4?, add link) or even improve the existing one (link here x2) so this is something that affects more cases than GIS.

## PG 12 changes

In the same blog that I mentioned about different strategies (link), Paul Ramsey (co-founder of the project) also mentioned that a posible solution for us would be to avoid doing all the work to only read a tiny fraction of the data, and instead have PostgreSQL support partial decompression and that's exactly what he proposed for PG12 (link to patch / discussion).


## PostGIS 3.1 changes

As PG was exposing an improved API but the existing code from PostGIS had been developed before that was a thing, so what now was missing was to find the places where using partial decompression was more efficient than reading the whole value. The main affected functions where:

* [Bounding box operators](https://postgis.net/docs/reference.html#idm9874) like [&&](https://postgis.net/docs/geometry_overlaps.html). Note that only affects when an index **isn't used** as indices store the bounding box separately to avoid reading the data itself.
* Functions that only need to read the headers: [`ST_SRID`](https://postgis.net/docs/ST_SRID.html),  [`ST_NDims`](https://postgis.net/docs/ST_NDims.html), `postgis_hasbbox`(link), `SE_Is3D`(link), `SE_IsMeasured`(link).

* As part of the PR but unrelated to decompression, casting a PostGIS point to a PostgreSQL point (link) was also improved by using operations over serialized data directly.


This was a manual work guided by experience, so it's likely that other functions could still benefit from a similar patch in the future. In fact I several functions like `ST_NPoints` or `ST_NRings` (LINKS) could be rewritten to read data directly from the headers but I


As an extra note, a few months after integrating the change I detected a performance regression over small geometries. Whereas the old API in those cases returned the buffer itself, the partial decompression one returned a copy of the buffer, so calling the new API was having a performance penalty due to the extra copy. Once detected the fix was simple: use the new API only when decompression is necessary, and keep the buffer as-is when it isn't (link change here).


## Performance comparison

I'll be again using the [Boundaries of Canada Provinces](https://open.canada.ca/data/en/dataset/a883eb14-0c0e-45c4-b8c4-b54c4a819edb) dataset:

```bash
$ shp2pgsql -D -s 4326 -I lpr_000b16a_e/lpr_000b16a_e canada | psql -U postgres benchmarks
```

Show size in disk (per geometry => toasted => needs decompression).


### Queries

To run the tests I'm using pgbench and running one query over and over for 30 seconds and comparing the query latency between PostGIS 3.0 and 3.1.

```sql
$ cat large.pgbench
EXPLAIN (ANALYZE, TIMING OFF) SELECT ST_SRID(geom) FROM canada; -- ST_SRID
--EXPLAIN (ANALYZE, TIMING OFF) SELECT ST_NDims(geom) FROM canada; -- ST_NDims
--EXPLAIN (ANALYZE, TIMING OFF) SELECT * FROM canada WHERE geom && ST_TileEnvelope(3,1,1); -- Reading a tile
--EXPLAIN (ANALYZE, TIMING OFF) SELECT a.gid, b.gid, a.geom && a.geom FROM canada a, canada b; -- Large geom &&
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
| Reading a tile | 119.533 | 8.906 | **13.421x** |
| Large geom `&&` | 3189.462 | 189.105 | **16.866x** |

Note that in these cases the improvement should be bigger the faster the disk are (as the decompression would be taking a bigger slice of the time). For example, when I tested the improvement I was seeing a **120 to 470x** improvement, but I was testing that in a PC with better specs back then and now I'm using a laptop with a much worse disk and constant jumps in CPU frequency. Or at least I hope that's the reason for big differences both in the 3.0 and 3.1 tests.

Although this is "just" a corner case, it's nice to see PostGIS improving the peformance around large geometries. Hopefully this will help reduce the amount of situations where people need to duplicate a geometry to work at different zoom levels.
