# smo_os_bng_grids

Pure Ruby gem for working with **Ordnance Survey British National Grid (BNG)** squares.
Point lookup, spatial search, bounds, corner coordinates, and Shapefile export — no external dependencies.

Developed by **Sebastian Madrid Ontiveros**.
Contains OS data. Crown copyright and database right 2025.
Licensed under the [Open Government Licence v3.0](https://www.nationalarchives.gov.uk/doc/open-government-licence/version/3/).

[![Gem Version](https://badge.fury.io/rb/smo_os_bng_grids.svg)](https://badge.fury.io/rb/smo_os_bng_grids)

---

## Features

- **Hardcoded geometry** sourced directly from the OS BNG Grids GeoPackage (EPSG:27700)
- Grid squares at **100km, 50km, 10km, 5km, and 1km** resolutions (910,091 squares total)
- **Point lookup** — find the grid ref containing any easting/northing at any resolution
- **Spatial search** — find all tiles intersecting a radius or bounding box around a point
- **Bounds** — retrieve min/max easting/northing for any grid ref
- **Corner points** — NW → NE → SE → SW → NW, ready for InfoWorks ICM `boundary_array`
- **Shapefile export** — pure Ruby SHP/SHX/DBF/PRJ writer, no GDAL required
- **Pure Ruby stdlib only** — works in InfoWorks ICM 2027 embedded Ruby

---

## Installation

```ruby
gem "smo_os_bng_grids"
```

Or install directly:

```bash
gem install smo_os_bng_grids
```

---

## Quick start

```ruby
require "smo_os_bng_grids"

lister = SmoOsBngGrids::Lister.new

# Point lookup — what grid square is Edinburgh city centre in?
easting  = 325000
northing = 673000

SmoOsBngGrids::Grid.ref_at(easting, northing, resolution: "10km")  # => "NT27"
SmoOsBngGrids::Grid.ref_at(easting, northing, resolution: "1km")   # => "NT2573"

# All resolutions at once
lister.find(easting, northing)
# => {"100km"=>"NT", "50km"=>"NTNW", "10km"=>"NT27", "5km"=>"NT27SE", "1km"=>"NT2573"}

# Bounds of a grid square
SmoOsBngGrids::Grid.bounds("NT27")
# => {min_e: 320000, min_n: 670000, max_e: 330000, max_n: 680000}
```

---

## Grid resolutions

| Resolution | Count  | Example ref | Square size     |
|-----------|--------|-------------|-----------------|
| 100km     | 91     | `NT`        | 100km × 100km   |
| 50km      | 364    | `NTNW`      | 50km × 50km     |
| 10km      | 9,100  | `NT27`      | 10km × 10km     |
| 5km       | 36,400 | `NT27SE`    | 5km × 5km       |
| 1km       | 910,000| `NT2573`    | 1km × 1km       |

---

## Listing grid squares

```ruby
lister = SmoOsBngGrids::Lister.new

# All 100km squares
lister.list("100km")

# All 10km squares within NT
lister.list("10km", within: "NT")

# All 1km squares within NT27
lister.list("1km", within: "NT27")
```

Each entry is a Hash:

```ruby
{
  ref:    "NT27",
  min_e:  320000, min_n: 670000,
  max_e:  330000, max_n: 680000,
  points: [
    [320000, 680000],  # NW
    [330000, 680000],  # NE
    [330000, 670000],  # SE
    [320000, 670000],  # SW
    [320000, 680000]   # NW (closed)
  ]
}
```

---

## Spatial search

Find all tiles intersecting a radius or bounding box around a point:

```ruby
lister = SmoOsBngGrids::Lister.new

# All 10km tiles within 12km of Edinburgh
tiles = lister.search(325000, 673000, resolution: "10km", radius: 12000)
tiles.each { |t| puts "#{t[:ref]}  #{t[:distance_m]} m" }

# All 1km tiles within 1.5km of Edinburgh
lister.search(325000, 673000, resolution: "1km", radius: 1500)

# All 5km tiles within a 15km box around Edinburgh
lister.search(325000, 673000, resolution: "5km", box: 15000)
```

Radius search also returns `:distance_m` — `0.0` when the point is inside the tile.

---

## Corner points — InfoWorks ICM

Every entry returned by `list`, `search`, and `entry_for` includes a `:points` array in **NW → NE → SE → SW → NW** order, matching the InfoWorks ICM `boundary_array` convention:

```ruby
lister = SmoOsBngGrids::Lister.new

# Single tile containing a point
tile = lister.search(325000, 673000, resolution: "10km", radius: 1).first
boundary_array = tile[:points]
# => [[320000, 680000], [330000, 680000], [330000, 670000], [320000, 670000], [320000, 680000]]

# Flat XY array if needed
flat_xy = tile[:points].flatten
# => [320000, 680000, 330000, 680000, 330000, 670000, 320000, 670000, 320000, 680000]

# Convert a ref string to a full entry
entry = lister.entry_for("NT27SE")
entry[:points]  # => [[325000, 675000], [330000, 675000], [330000, 670000], [325000, 670000], [325000, 675000]]
```

---

## Shapefile export

Export any set of entries to ESRI Shapefile format (OSGB36 / EPSG:27700).
Pure Ruby — no GDAL, no external gems, works in InfoWorks ICM 2027.

```ruby
lister = SmoOsBngGrids::Lister.new
writer = SmoOsBngGrids::ShapefileWriter.new

# From list()
writer.write(lister.list("10km", within: "NT"), "/tmp/nt_10km")

# From search()
entries = lister.search(325000, 673000, resolution: "10km", radius: 20000)
writer.write(entries, "/tmp/edinburgh_10km_20km")

# From find() — one tile per resolution
found = lister.find(325000, 673000)
found.each do |res, ref|
  writer.write([lister.entry_for(ref)], "/tmp/edinburgh_#{res}")
end
```

Produces `.shp`, `.shx`, `.dbf`, `.prj` — open directly in QGIS or ArcGIS.

---

## Grid reference validation

```ruby
SmoOsBngGrids::Grid.valid?("NT")      # => true
SmoOsBngGrids::Grid.valid?("NT27")    # => true
SmoOsBngGrids::Grid.valid?("NT2573")  # => true
SmoOsBngGrids::Grid.valid?("ZZ")      # => false
```

---

## Data

All geometry is hardcoded from the **OS BNG Grids GeoPackage** published by Ordnance Survey.
Source: [github.com/OrdnanceSurvey/osbng-grids](https://github.com/OrdnanceSurvey/osbng-grids)

Contains OS data. Crown copyright and database right 2025.
Licensed under the Open Government Licence v3.0.

---

## Attribution

Built by **Sebastian Madrid Ontiveros** to support hydraulic modelling and flood risk workflows in the UK.
If this gem saves you time, consider [buying Sebastian a coffee](https://buymeacoffee.com/smadrid).
