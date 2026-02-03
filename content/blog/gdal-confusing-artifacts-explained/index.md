---
title: "GDAL Pitfalls Explained: Confusing Tools, Hidden Behaviors, and How to Test Them"
date: 2026-02-03T12:30:00+03:00
draft: false
type: post
slug: gdal-confusing-artifacts-explained
categories: ["GIS", "Remote Sensing"]
tags: ["Raster Data", "GDAL", "Geospatial Data"]
image:
  filename: GDAL_cover.png
  alt_text: "Geospatial Data Abstraction Library"
build:
  render: always
  list: local
---

### TL;DR

GDAL has many tools with similar names that behave very differently.
Using the wrong one often does not fail loudly — it silently produces misaligned rasters, broken tiles, or inefficient cloud pipelines.

This article explains the most commonly confused GDAL tools (such as gdal_translate vs gdalwarp, gdal2tiles.py vs gdal_retile.py, and others), shows minimal reproducible tests, and gives clear rules of thumb so you can choose the right tool with confidence.


### Why GDAL feels confusing (and why that’s not your fault)
GDAL is powerful, battle-tested, and old.

Many of its command-line tools were created at different times to solve different problems, long before:

- cloud-native raster access

- tile servers

- Cloud Optimized GeoTIFFs (COGs)

- interactive web maps

As a result:

- tools with similar names do very different things

- assumptions are implicit, not enforced

- incorrect usage often succeeds without warnings

The problem isn’t that GDAL is poorly designed — it’s that modern workflows expose sharp edges that used to stay hidden.

This post focuses on those edges.

### How this article is structured

For each confusing GDAL artefact, I follow the same pattern:

1. What people assume

2. What actually happens

3. A minimal reproducible test

4. How to verify the result

5. A practical rule of thumb

All examples are designed to be tested locally with gdalinfo, rio info, visual inspection, or tile requests.

### 1. gdal_translate vs gdalwarp

#### What people assume

Both “convert” rasters

`gdal_translate` can reproject data

What actually happens

`gdal_translate` operates on pixels

`gdalwarp` operates on geometry

gdal_translate will never change:

- CRS

- geotransform

- pixel alignment

gdalwarp always can.

#### Reproducible test

```bash
# Input raster in EPSG:4326
gdalinfo input_4326.tif | grep "Coordinate System"

# Translate (format-only)
gdal_translate input_4326.tif out_translate.tif

# Warp (reproject)
gdalwarp -t_srs EPSG:3857 input_4326.tif out_warp.tif

```

#### Verification
Verification

- `out_translate.tif` remains EPSG:4326

- `out_warp.tif` is EPSG:3857

- Bounding boxes differ

- Pixel sizes differ

Rule of thumb

If coordinates change, `gdal_translate` is the wrong tool.

### 2. gdal2tiles.py vs gdal_retile.py
What people assume
- Both "make tiles"

#### What actually happens
These tools solve completely different problems.

| Tool             | Purpose                         |
| ---------------- | ------------------------------- |
| `gdal2tiles.py`  | Web map tiles (XYZ / TMS)       |
| `gdal_retile.py` | Chunking rasters for processing |

`gdal2tiles.py` produces:

- PNG/JPEG tiles

- zoom pyramids

- web-map-ready output

gdal_retile.py produces:

- GeoTIFF chunks

- same CRS and resolution

- processing-oriented outputs

#### Reproducible test

- Run both tools on the same raster

Attempt to:

- load results in QGIS

- serve them via a tile server

Verification

`gdal2tiles` output has no geotransform

`gdal_retile` output is not web-tile-aligned

Rule of thumb

`gdal_retile` is never a substitute for web tiles.

### 3. gdal_merge.py vs gdalwarp mosaic

What people assume

- “Merge” means mosaic

#### What actually happens

gdal_merge.py:

- assumes perfect alignment

- does not reproject

- does not resample

- does not validate grids

gdalwarp:

- reprojects

- resamples

- aligns pixels

- handles overlaps explicitly

#### Reproducible test

Mosaic two rasters with:

- slightly different CRS

- slightly different resolution


```bash
gdal_merge.py -o merged.tif a.tif b.tif
gdalwarp a.tif b.tif warped_mosaic.tif

```

#### Verification

- Visual seams

- Misaligned pixels

- Silent artefacts in gdal_merge.py

Rule of thumb

`gdal_merge.py` assumes ideal data. Real data isn’t ideal.

### 4. Overviews: gdaladdo vs the COG driver

#### What people assume

Once built, overviews are preserved

#### What actually happens
Overviews are often
- dropped during build
- rebuilt incorrectly
- stored externally

Unless explicitly handled at the final output stage.

#### Reproducible test

```bash
gdaladdo input.tif 2 4 8 16
gdal_translate input.tif out_normal.tif
gdal_translate input.tif out_cog.tif -of COG

```

#### Verification
```js
rio cogeo validate out_normal.tif
rio cogeo validate out_cog.tif
```

#### Rule of thumb
Build overviews at the final output stage, not the first.

### 5. -tr, -ts, and -tap (silent grid killers)
This is one of the most common causes of misaligned tiles.

#### What people assume
- Resolution flags are interchangeable

#### What actually happens
| Flag   | Meaning              |
| ------ | -------------------- |
| `-tr`  | target resolution    |
| `-ts`  | target image size    |
| `-tap` | align pixels to grid |


Without -tap, GDAL may:

- shift pixel origins

- introduce half-pixel offsets

- break tile alignment

#### Reproducible test
Warp the same raster three ways
1. `-tr` only

2. `-ts` only

3. `-tr` + `-tap`

#### Verification

- Compare pixel origins

- Request adjacent tiles

- Look for seams

#### Rule of thumb
Misaligned rasters don’t crash — they poison everything downstream.

### 6. NoData vs zero values

#### What people assume

- Zero means empty

#### What actually happens

Zero is often a valid measurement.

Tile servers and renderers treat:

- NoData == transparent 
- Zero == data

Confusing the two causes:

- disappearing features

- incorrect statistics

- broken color ramps


#### Reproducible test

- One raster with real zeros
- One raster with NoData
- Serve them via a tile server

#### Verification

- Histogram differences
- Transparency behaviour
- Edge artefacts

#### Rule of thumb

NoData is semantic, not numeric.

### 7. Scale / offset vs physical rescaling
#### What people assume
- Applying -scale reduces file size

#### What actually happens
Scelte/offset:
- changes metadata
- does not change storage
- does not reduce IO cost

Real optimization requires
- datatype conversion
- quantization
- compression

#### Reproducible test

- Apply scale only
- Convert to UInt16
- Compare file size and latency

Rule of the thumb

Metadata tricks do not optimize delivery

### 8. "COG-like" GeoTIFF vs true COGs
#### What people assume
- Overviews = COG

#### What actually happens
True COGs require:
- Correct internal tiling
- Correct overview layout
- Correct byte ordering

Many "COG-like" files fails under HTTP range requests.

#### Verification

`rio cogeo validate file.tif`

#### Rule of thumb
If it is not validated it is not a COG


### How all examples were tested

All examples were tested using:

- `gdalinfo`

- `rio info`

- visual inspection in QGIS

- tile requests via a tile server

The focus was on:

- CRS integrity

- pixel alignment

- overview behavior

- real-world downstream effects

#### A mental model that actually works

Think of GDAL tools in three layers:

1. Geometry → gdalwarp

2. Pixels → gdal_translate

3. Distribution → COGs, tile servers

If a command crosses layers, stop and verify.

Final checklist

Before trusting any raster output, ask:

- Did geometry change?

- Did pixel alignment change?

- Are overviews correct?

- Is NoData intentional?

- Is this optimized for access, not storage?

If you can’t answer all five, test again.

### Closing

GDAL is not confusing, unverified assumptions are.

Once you understand which tools touch geometry, which touch pixels, and which prepare data for distribution, most “mysterious” bugs stop appearing.

And when in doubt: test, don’t assume.


### Code and Outputs
All code and outputs are available on GitHub:
https://github.com/Thuhaa/gdal-confusing-artifacts
