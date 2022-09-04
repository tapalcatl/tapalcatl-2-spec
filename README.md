# Tapalcatl 2

Tapalcatl 2 is a formalization of using ZIP archives for storage of
tile-based data.

Justifications for the Tapalcatl 2 format are documented in [Tapalcatl:
cloud-optimized tile
archives](https://medium.com/@mojodna/tapalcatl-cloud-optimized-tile-archives-1db8d4577d92).

## Sample Metadata

Root-level metadata is a subset of [TileJSON 2.2.0] augmented with Tapalcatl
attributes. By convention, it is stored as `meta.json` in the root of a tree
of tiles. E.g.:

```javascript
{
  // Tapalcatl version
  "tapalcatl": "2.0.0",
  "name": "Land Cover",
  "description": "Unified land cover, derived from MODIS-LC, ESACCI-LC, NLCD, and C-CAP.",

  // overall tileset extent
  "minzoom": 0,
  "maxzoom": 7,
  "bounds": [
    -180,
    -85.0511287798066,
    180,
    85.0511287798066
  ],

  // formats present, mapping extension to Content-Type header
  "formats": {
    "png": "image/png"
  },

  // min/max scale of tiles; corresponds to @<scale>x filename component
  "minscale": 2,
  "maxscale": 2,

  // number of root-adjacent tiles included in an archive (with their respective
  // sub-pyramids)
  "metatile": 4,

  // zooms at which archives exist; tiles for intermediate zooms are contained
  // in the next lowest archive (i.e. zoom 5 tiles are in zoom 4 archives)
  "materializedZooms": [
    0,
    4
  ],

  // URL template for individual archives
  "source": "s3://mojodna-temp/lc-png/{z}/{x}/{y}.zip"
}
```

Archive-level metadata is stored as a ZIP comment within each archive. E.g.:

```javascript
{
  // root tile coordinates (typically corresponds to the archive's filename)
  "root": "4/0/0",

  // everything else: as above, but specific to this archive
  "tapalcatl": "2.0.0",
  "name": "Land Cover",
  "description": "Unified land cover, derived from MODIS-LC, ESACCI-LC, NLCD, and C-CAP.",
  "minzoom": 4,
  "maxzoom": 7,
  "bounds": [
    -180,
    66.51326044311186,
    -90,
    85.0511287798066
  ],
  "formats": {
    "png": "image/png"
  },
  "minscale": 2,
  "maxscale": 2,
  "metatile": 4
}

```

Let's break this down...

## Concepts

### Format

Tiles in multiple formats may be included in the same tileset. In practice,
this often means varying image formats (PNG, WEBP, JPEG, TIFF) or tiled
vector data serialized differently (GeoJSON, TopoJSON, Mapbox Vector Tiles).

Data being represented should be the same; file format should be the only
difference.

### Scale

Tiles at multiple scales may be included in the same tileset. In image terms,
a 256𝗑256 is scale `1` and would correspond to `<z>/<x>/<y>.png` within the
archive. At scale `2`, it would be 512𝗑512 and correspond to
`<z>/<x>/<y>@2x.png` within the archive.

### Metatiles

A metatile groups a set of physical tiles. For Tapalcatl 2, that means
grouping them into a single archive. Metatile size corresponds to the number
of tiles on a side, so the effective number of tiles covered by a metatile is
`2^<metatile>`.

Zoom level 0 contains a single tile, so `0/0/0.zip` will only contain `0/0/0`
and its sub-pyramids.

Zoom level 4 is a 16𝗑16 grid, corresponding to 256 tiles. With a metatile
size of 4, it will be broken up into 16 archives (`4/0/0`, `4/4/0`, `4/8/0`,
`4/12/0`, etc.), each containing 16 zoom 4 tiles (`4/0/0` contains
`4/{0..3}/{0..3}`) and their sub-pyramids.

Using metatiles reduces the number of objects required to store tiles at a
specific zoom.

Metatiles must be powers of 2 in order to align with the global grid system.

### Materialized Zooms

If metatiles group objects in 2 dimensions, materialized zooms introduce a
3rd by allowing sub-pyramids to be included in the same archive (which
exponentially decreases the number of objects needing to be managed). A list
of materialized zooms indicates which zoom levels archives are expected to
exist at ("be material").

For a tileset covering zooms 0-7, if it is materialized at zooms 0 and 4, no
archives will exist at any other zooms. Thus, `0/0/0.zip` will contain
`0/0/0` and all descendents up to zoom 4. `4/0/0.zip` will contain `4/0/0`
and all of its descendents (as well as its metatiles and their descendents).

The tileset's root (`0/0/0` for global data) will always correspond to a
materialized zoom. If no other materialized zooms are provided, `0/0/0.zip`
will be the sole archive, containing the full tileset.

## Properties

### minzoom

Minimum zoom level contained in a tileset / archive. When associated with a
tileset, it applies to all archives. When associated with an archive, it only
applies to tiles contained within that archive.

### maxzoom

Maximum zoom level contained in a tileset / archive. When associated with a
tileset, it applies to all archives. When associated with an archive, it only
applies to tiles contained within that archive.

### bounds

Geographic extent of a tileset / archive. When associated with a tileset, it
applies to all archives. When associated with an archive, it only applies to
tiles contained within that archive.

### formats

A mapping between filename extension and `Content-Type` header. Entries must
exist for each format included in an archive. If additional HTTP headers
should be included, replace the `Content-Type` string with a mapping between
header names and values, e.g. for gzip-compressed Mapbox Vector Tiles:

```javascript
{
  // ...
  "formats": {
    "mvt": [
      {
        "Content-Type": "application/vnd.mapbox-vector-tile"
      },
      {
        "Content-Encoding": "gzip"
      }
    ]
  }
}
```

### minscale

Minimum scale if tiles at multiple (or non-`1`) scales are present in a
tileset. Can be assumed to be `1` if absent.

### maxscale

Maximum scale if tiles at multiple (or non-`1`) scales are present in a
tileset. Can be assumed to be `1` if absent.

### metatile

Metatile size. See [Metatile](#metatiles) for more discussion.

### materializedZooms (tileset)

List of materialized zooms. See [Materialized Zooms](#materialized-zooms) for
more discussion.

### source (tileset)

URL template to map archive coordinates to objects. `{z}`, `{x}`, and `{y}`
are required. `{h}` indicates that a 5 character hash should be calculated
from the tile coordinates (`md5("<z>/<x>/<y>").slice(0, 5)`) and substituted.

Including `source` allows metadata to be hosted elsewhere from archives.

### root (archive)

Root tile coordinates for items contained in the archive. This _should_ match
the archive's filename (and can be determined from the archive's contents).

## Converting between tile coordinates and archive coordinates

### JavaScript

The following converts a tile coordinate (`6/10/23`) to its material
coordinate (`4/2/5`) and the metatile coordinate (`4/0/4`), which correspond
to the archive containing data:

```javascript
const zoom = 6;
const x = 10;
const y = 23;
const materializedZooms = [0, 4];
const metatile = 4;

// calculate source archive coordinates
const mz = materializedZooms
  // prevent mutation by .reverse()
  .slice()
  .reverse()
  // find the next lowest materialized zoom
  .find(z => z <= zoom); // => 4

const dz = zoom - mz; // => 2

// find ancestor coordinates at target (materialized) zoom
let mx = x >> dz; // => 2
let my = y >> dz; // => 5

// convert to metatile coordinate
mx -= mx % metatile; // => 0
my -= my % metatile; // => 4
```

## License

This specification is made available under the terms of the [Creative Commons Zero license](http://creativecommons.org/publicdomain/zero/1.0/).
