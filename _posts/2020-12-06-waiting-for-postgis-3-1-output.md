---
layout: post
title: "Waiting for Postgis 3.1: Output functions"
subtitle: 'On the improvements to PostGIS output functions'
tags: [postgis, performance]
comments: false
---

To continue with the changes in [PostGIS 3.1](http://postgis.net/2020/11/19/postgis-3.1.0alpha3/), in this post I'll cover the performance improvements on many functions that output geometries either as binary or as text. I will talk about several changes which have in common that they kicked off by a single question: "Now what?"

After the release of 3.0 back in October 2019, I wasn't working on anything directly related to PostGIS with the exception of using MVTs to make maps. Since I had recently improved vector tile functions and I was ok with their performance (_spoiler alert_: [I ended up improving them again in 3.1](https://rmr.ninja/2020-11-19-waiting-for-postgis-3-1-mvt/)) I didn't have any good ideas about what to work on next. But then two events happened around the same time: [CppCon](https://cppcon.org/) released their 2019 talks on YouTube and a coworker mentioned moving large amounts of information between PostgreSQL and BigQuery, which was not only slow but also prone to add inaccuracies to the data. 

It turned out that the only way there was to avoid BigQuery altering the geometries on input [was to use the GeoJSON](https://cloud.google.com/bigquery/docs/gis-data#loading_geojson_data) format ([ST_AsGeoJSON](https://postgis.net/docs/ST_AsGeoJSON.html)) which outputs a JSON with the geometry in its text form, and I had [just watched](https://www.youtube.com/watch?v=4P_kbF0EbZM) Stephan T. Lavavej talk where he explained how they had improved the performance of the conversion from floating point numbers to string in Microsoft's [C++ Standard Library](https://github.com/microsoft/STL). Those two things clicked in my head: If I could apply a similar approach inside PostGIS, we would print geometries 10x faster. Once I started exploring the code and deciding how to best approach the issue I also found several quick enhancements that could be both to text and binary output functions so the task grew but the spirit remained: Let's make getting geometries out of PostGIS faster.

## Floating point to string

The foundation on which Microsoft's \<charconv\> change rested was **Ryū**, an [algorithm](https://github.com/ulfjack/ryu) developed by Ulf Adams that greatly improved the speed of float to string conversion, which is likely to be one of the most used functions in computer programs. Think about how many times we print numbers every day: anything from logs, reports, showing them on screen, ETLs... Improving  a function like this means a direct reduction in the energy we use on our devices and data centers, which means that it literally avoids the emission of tons of CO2, which reduces the impact of mankind on climate, slightly tipping the scale on our favor in the fight to save the world from ourselves.

Most developers don't need to know how to convert a floating point number to string since the standard libraries or the programming languages themselves give us that functionality, and in an ideal world the enhancements introduced by Ryū would be automagically implemented everywhere solving all of our problems. Sadly for us the reality is different and the changes in the output (when compared with the traditional `printf` output) and the extremely long process to update core system libraries means that there is always more work to do. So although not every programmer has to care about this kind of stuff, some do. For example, Andrew Gierth (a.k.a. our beloved [RhodiumToad](http://blog.rhodiumtoad.org.uk/) on IRC) brought Ryū's improvements to PostgreSQL 12 and I set up to emulate him and do the same in PostGIS.

### First implementation: A hack

The first step in the process was to confirm the estimation of how impactful the change would be. I wasn't looking for a perfect match, instead I just wanted to get something that was close enough to the existing coordinate output with a much better performance.

Since PostgreSQL 12 had introduced a similar change, I modified PostGIS' [`lwprint_double`](https://github.com/postgis/postgis/blob/d1a410ac26b96d325aebc5915fd5a591da5ba934/liblwgeom/lwprint.c#L455) to use those functions directly with an ugly hack that involved hardcoding compiler and linker flags relative to my local installation. This confirmed that I was in the correct path as I saw a **3-4x** improvement in performance in `ST_AsText`. From that moment, I knew that working on a clean integration was indeed worth it.

### Second implementation: Ryū

With the proven thesis I moved on to integrate the library into PostGIS itself. In the proposed change ([#523](https://github.com/postgis/postgis/pull/523/)) I used Ryū's `d2fixed_buffered_n` and `d2exp_buffered_n` to print coordinates and it worked great: depending on the function (and what percentage of the CPU time was actually spent printing doubles) they became **1.2x to 8x** as fast as they were in 3.0. Nevertheless I did find several points of friction when trying to match the previous output:

* It printed as many decimal digits as required by the caller. This was in part because it was hard to know how many digits you needed to ensure the number you printed would be reimported as exactly the same binary number, and in many occasions it lead to having a lot of meaningless data: for example, `0.3` with precision `40` becomes `0.2999999999999999888977697537484345957637`.

* The previous implementation had some bugs with the precision parameter (which determines how many **decimal** digits are included) and sometimes returned less characters than requested. Those issues had to be addressed.

* I couldn't simply replace the old call to `sprintf` with Ryu's functions because 3.0 did extra operations like trimming trailing zeros. I initially added this truncation to our version of Ryū but I wasn't very happy with the change since I was adding characters to the buffer to remove them moments later.

* Ryū's `d2fixed_buffered_n` doesn't provide a way to limit the precision of the output. My initial implementation truncated the string result, which was slow and wrong as it didn't do proper rounding.

With all this in mind I decided that I would rather break compatibility and change the coordinate output format to provide a faster and more human friendly alternative. Hopefully with less bugs.


### Third implementation: Custom Ryū

The final step, which will be part of PostGIS 3.1, was focused on getting a better coordinate output while keeping the performance improvements of the second iteration. After multiple tests and discussions, the final decision was to use the following rules:

* Use the **shortest representation**, which is enough to guarantee round-trip safety. `0.3` will always be represented as `0.3` for any precision greater than 0.

* Use scientific notation for absolute numbers smaller than `1e-8`. The previous behaviour was to output `0` for absolute values smaller than `1e-12`, which meant a precision loss around zero.

* Use scientific notation for absolute numbers greater than `1e+15`, which was the same behaviour as before.

* The precision parameter still limits only the number of decimal digits of the output but now it is applied with proper rounding to the shortest representation, that is it will only trim **meaningful** decimal digits. It will also be applied exactly in the same way to all text output functions.

* The precision parameter now also affects the scientific notation too, whereas before the precision for large numbers was fixed to between 5 and 8 digits.

* The default precision value remains unchanged: `9` for GeoJSON and `15` for everything else.

The [new code](https://github.com/postgis/postgis/pull/570) is based on Ryū's `d2s` and modified to handle the format defined above and has a faster and more consistent output than before. Let's see an example of the same geometry at different precision levels:

#### Postgis 3.0:

```sql
SELECT x, ST_AsText(ST_MakePoint(0.3, 22.200000000000003), x)
FROM generate_series(1, 20, 2) x;
 x  |                     st_astext                     
----+---------------------------------------------------
  1 | POINT(0.3 22.2)
  3 | POINT(0.3 22.2)
  5 | POINT(0.3 22.2)
  7 | POINT(0.3 22.2)
  9 | POINT(0.3 22.2)
 11 | POINT(0.3 22.2)
 13 | POINT(0.3 22.2)
 15 | POINT(0.3 22.2)
 17 | POINT(0.29999999999999999 22.200000000000003)
 19 | POINT(0.2999999999999999889 22.20000000000000284)
(10 rows)
```

#### Postgis 3.1:

```sql
SELECT x, ST_AsText(ST_MakePoint(0.3, 22.200000000000003), x)
FROM generate_series(1, 20, 2) x;
 x  |           st_astext           
----+-------------------------------
  1 | POINT(0.3 22.2)
  3 | POINT(0.3 22.2)
  5 | POINT(0.3 22.2)
  7 | POINT(0.3 22.2)
  9 | POINT(0.3 22.2)
 11 | POINT(0.3 22.2)
 13 | POINT(0.3 22.2)
 15 | POINT(0.3 22.200000000000003)
 17 | POINT(0.3 22.200000000000003)
 19 | POINT(0.3 22.200000000000003)
(10 rows)
```

We can see the two most noticeable changes here:

* `0.29999999999999999` represents the same binary value as `0.3`, so in 3.1 we always prefer `0.3` as it's sorter and still safe for a round trip.

* At precision 15, you already have enough digits to show `22.200000000000003` which, but due to a bug, wouldn't show until precision 17 in 3.0. As it's the case for `0.3`, that number already has as many digits as needed to uniquely identify a binary floating point number, so there is no need to add more digits in higher precision levels.

## Other changes

Once you speed up the slowest wheel in the process others raise in importance or even become the new bottleneck, so aside from introducing Ryū and changing the coordinate output format I also applied several other performance improvements:

* In many output functions, both in text and binary output, we now generate the exact buffer that we are going to return to PostgreSQL instead of a temporary one that later needs to be copied to add a header ([#541](https://github.com/postgis/postgis/pull/541)).

* We no longer use intermediate buffers when printing individual coordinates ([#573](https://github.com/postgis/postgis/pull/573)) to avoid `memcpy` and `strlen` calls ([#537](https://github.com/postgis/postgis/pull/537), [r8792203](https://github.com/postgis/postgis/commit/87922037fd09b68ad2ada409f776e06590924256)).

* In functions that need the SRS for the output (like ST_AsGML or optionally ST_AsGeoJSON) we cache it instead of generating it for each row ([#557](https://github.com/postgis/postgis/pull/557)). We also avoid SQL inlines where the cache is destroyed after each row, which made it useless ([#561](https://github.com/postgis/postgis/pull/561)). This also affects [ST_GeomfromGeoJSON](https://postgis.net/docs/ST_GeomFromGeoJSON.html) and the the equivalent `'{}'::geometry`.

* I also adapted the cost of the SQL functions to help PostgreSQL planner make better decisions ([#556](https://github.com/postgis/postgis/pull/556)).


## Performance comparison

For most of the benchmarks I'll use the [Boundaries of Canada Provinces](https://open.canada.ca/data/en/dataset/a883eb14-0c0e-45c4-b8c4-b54c4a819edb) dataset, which contains only 13 multipolygons with an average of 260k points and over 2000 rings. It's my favourite dataset when I want to do any performance test.

```bash
$ shp2pgsql -D -s 4326 -I lpr_000b16a_e/lpr_000b16a_e canada | psql -U postgres benchmarks
```

For ST_GeoHash, which only works with points, I'll use the [2015 NYC tree census](https://data.cityofnewyork.us/Environment/2015-Street-Tree-Census-Tree-Data/pi5s-9p35) with 683788 points.

```bash
$ shp2pgsql -D -s 4326 -I geo_export_d02b464e-5a77-4dc8-b5d1-f92150d03a11 trees | psql -U postgres benchmarks
```

And for ST_AsEncodedPolyline, which only works on single LineStrings, I'll use the first line of each of the 4248 MultiLineStrings of the [US coastile from Tiger 2019](https://www2.census.gov/geo/tiger/TIGER2019/COASTLINE/).

```bash
$ shp2pgsql -D -s 4326 -I tl_2019_us_coastline coastline | psql -U postgres benchmarks
```

To run the tests I'm using pgbench and running one query over and over for 30 seconds and comparing the query latency between PostGIS 3.0 and 3.1.

```
$ head -n 5 file.pgbench 
-- Binary output functions
EXPLAIN (ANALYZE , TIMING OFF) SELECT ST_AsBinary(geom) FROM canada;
--EXPLAIN (ANALYZE , TIMING OFF) SELECT ST_AsTWKB(geom) FROM canada;
--EXPLAIN (ANALYZE , TIMING OFF) SELECT ST_GeoHash(geom) FROM trees;
--EXPLAIN (ANALYZE , TIMING OFF) SELECT ST_AsEncodedPolyline(geom) FROM coastline_simple;
$ pgbench -c 1 -T 30 -r -f file.pgbench -U postgres benchmarks
...
statement latencies in milliseconds:
       154.625  EXPLAIN (ANALYZE , TIMING OFF) SELECT ST_AsBinary(geom) FROM canada;
```

### Binary output functions

These functions were the least impacted by the changes as they were only affected by some of the minor improvements to avoid unnecessary copies. Nevertheless, we see an improvement of **4-9%**;

| Function | 3.0 latency (ms) | 3.1 latency (ms) | Change |
| :------ |:--- | :--- | :--- |
[ST_AsBinary](https://postgis.net/docs/ST_AsBinary.html) (polygons) | 154.681 | 142.434 | **1.09x** |
[ST_AsTWKB](https://postgis.net/docs/ST_AsTWKB.html) (polygons) | 236.913  | 226.418 | **1.05x** |
[ST_GeoHash](https://postgis.net/docs/ST_GeoHash.html) (points) | 687.486 | 634.628 | **1.08x** |
[ST_AsEncodedPolyline](https://postgis.net/docs/ST_AsEncodedPolyline.html) (lines) | 207.756 | 199.336 | **1.04x** |

### Text output functions

In text output functions, we see the full impact of all the changes with functions that are **2 to 40x** as fast as the previous release.

| Function | 3.0 latency (ms) | 3.1 latency (ms) | Change |
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
