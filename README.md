# Mapping Tamil Nadu in QGIS: A Practical Walkthrough of Georeferencing, Digitization, and Thematic Cartography

*A hands-on QGIS 3.28 workflow — from a scanned paper map to a published set of thematic maps — built using Census of India 2011 data for Tamil Nadu.*

---

## Why this workflow matters

A huge share of real-world GIS work isn't exotic modeling — it's turning a scanned paper map or a messy spreadsheet into something spatially usable. This walkthrough covers exactly that pipeline in QGIS: georeferencing a raster scan, digitizing administrative boundaries from scratch, joining tabular census data to those boundaries, and producing publication-ready thematic maps and spatial queries. Every step below was run end-to-end on a Tamil Nadu district map and Census 2011 data.

**Tools used:** QGIS 3.28 (Firenze), Georeferencer GDAL plugin, OpenStreetMap XYZ tiles, Census of India 2011 district-level data.

---

## 1. Georeferencing a scanned map

Before a scanned map (a JPG or TIFF with no spatial reference) can be used in GIS, it needs to be tied to real-world coordinates. QGIS handles this through the **Georeferencer** plugin (`Raster → Georeferencer`).

The process is straightforward once you know the pattern:

1. Load the scanned raster into the Georeferencer window.
2. Identify points on the image where a coordinate grid is visible — usually meridian/parallel intersections printed on the source map.
3. Use **Add Point**, enter the known X (longitude) / Y (latitude) for each point, and repeat for at least 4 points spread across the image. More ground control points (GCPs) generally means better accuracy.
4. Under **Transformation Settings**, choose a transformation type (Polynomial 1 for a simple linear warp), set the target CRS to `EPSG:4326 – WGS 84`, and specify an output raster.
5. Run **Start Georeferencing**. QGIS warps the raster and loads the georeferenced version directly onto the map canvas — now aligned to its real-world location.

The residual values shown in the GCP table are worth watching: high residuals mean a point was clicked imprecisely, and nudging those points slightly tightens the fit.

![Georeferenced map of Tamil Nadu, correctly aligned over its real-world coordinates](images/01-georeferenced-tamilnadu.png)
*Figure 1 — The scanned Tamil Nadu district map after georeferencing, shown correctly positioned in WGS 84.*

---

## 2. When there's no coordinate grid: rubber sheeting

Not every scanned map has a printed coordinate grid to reference against. For those cases, QGIS supports **rubber sheeting** (image-to-image registration) — georeferencing one image against another image that's *already* spatially aware, like an OpenStreetMap basemap.

The workflow mirrors standard georeferencing, with one key difference: instead of typing in coordinates manually, you pick a recognizable feature on the scanned image (a coastline bend, a town, a river confluence), then use **From Map Canvas** to click the same feature on the live OSM basemap. QGIS reads off the coordinates automatically. Repeating this for 4–5 well-distributed points and running the transformation produces a warped image that now snaps correctly onto the real map — even though nothing on the original scan carried coordinate information.

---

## 3. Digitizing vector features from the raster

With a georeferenced raster as a backdrop, the next step is **digitization** — tracing real features (district boundaries, roads, points of interest) into proper vector layers.

**For polygons (district boundaries):**
- Create a new Shapefile layer (`Layer → Create Layer → New Shapefile Layer`), choosing Polygon geometry and defining attribute fields (district name, ID, etc.).
- Enable **Toggle Editing**, then use **Add Feature** to trace each district outline directly over the georeferenced backdrop.
- The **Split Features** tool is particularly useful here: rather than manually tracing every internal district border, you can digitize the outer state boundary once and then use Split Features to carve internal districts out of it — starting and ending each cut outside the polygon being split.

**For lines (road networks):**
- A new Line-geometry shapefile layer is created the same way.
- With Toggle Editing and **Add Line Feature** active, roads are traced with left-clicks along the route and a right-click to close each segment, assigning an ID to each completed road via the attribute dialog.

Common digitization errors to watch for — dangles (lines that don't quite connect), overshoots/undershoots, and slivers (tiny gaps between adjacent polygons that should share a border) — are worth checking for before finalizing any layer, since they silently break topology in later analysis.

Once digitized, a proper cartographic layout (via `Project → New Print Layout`) adds a scale bar, north arrow, coordinate grid, legend, and title before exporting the final map as an image or PDF.

![Administrative map layout of Tamil Nadu showing all 32 districts with grid, legend, and scale bar](images/02-administrative-map-layout.png)
*Figure 2 — Final administrative layout after digitizing district boundaries and adding cartographic elements.*

![Road network of Tamil Nadu digitized as line features over the district boundary layer](images/03-road-network-map.png)
*Figure 3 — Road network digitized as a separate line layer, styled and laid out with a legend and source note.*

---

## 4. Joining non-spatial data and building thematic maps

A shapefile of district boundaries is only half the picture — the real analytical value comes from attaching tabular data (population, literacy, etc.) to it.

**Joining tabular data:**
- Load the CSV/XLS census data as a layer.
- In the boundary layer's **Properties → Joins**, add a vector join specifying the join field (e.g., `district`) on both the boundary layer and the data table.
- The attribute table of the boundary layer now carries every joined column, ready to symbolize.

**Choropleth mapping:** In `Properties → Symbology`, choosing **Graduated** symbology on a numeric field (e.g., female literacy), selecting a class count and color ramp, produces a shaded map where color intensity communicates the underlying value across districts.

![Choropleth map of male literacy across Tamil Nadu districts, 2011](images/04-male-literacy-choropleth.png)
*Figure 4 — Graduated (choropleth) symbology showing male literacy by district.*

**Proportional pie charts:** Right-clicking the layer → **Properties → Diagrams** → Pie Chart lets you assign two or more numeric attributes (e.g., male and female population) as pie slices sized and colored per district, giving a compact multi-variable view overlaid directly on the map.

![Pie chart diagrams overlaid on district boundaries showing the male-female literacy split](images/05-male-female-literacy-piechart.png)
*Figure 5 — Proportional pie charts comparing male and female literacy by district.*

**Histograms:** The same Diagrams panel offers a Histogram option, useful for comparing magnitudes (e.g., total male vs. female population) as small bar charts positioned at each district's centroid.

![Histogram diagrams comparing male and female population across districts](images/06-male-female-population.png)
*Figure 6 — District-level histograms comparing male and female population counts.*

---

## 5. Spatial queries (attribute and location-based selection)

QGIS's selection tools go well beyond a basic "click to select." A few that are genuinely useful day-to-day:

- **Select Features by Polygon / Freehand / Radius** — draw a shape or set a radius directly on the canvas, and every feature intersecting it gets selected, with the attribute table highlighting in sync.
- **Select Features by Value** — filter by an exact or partial field value (e.g., all districts whose population field starts with "14").
- **Select Features by Expression** — the most flexible option, supporting SQL-like syntax:
  - `"Dis" LIKE 'P%'` — all districts starting with "P"
  - `"Dis" LIKE '_a%'` — districts with "a" as the second letter
  - `"TOT_P" > 3000000 AND "TOT_F" > 900000` — compound conditions combining multiple fields
  - Swapping `AND` for `OR` broadens the match to *either* condition being true

These same techniques scale directly to real analysis: filtering by population thresholds, isolating administrative units near a facility, or combining demographic and geometric criteria in one query.

**Extract by Location** (`Vector → Analysis/Selection Tools`) is worth calling out separately — it's how you pull, say, only the cities within Tamil Nadu out of a national cities dataset, using one layer's geometry to filter another.

---

## 6. Geoprocessing: buffer, clip, dissolve, union, convex hull

The final piece of the workflow is standard vector overlay analysis (`Vector → Geoprocessing Tools`):

| Tool | What it does |
|---|---|
| **Buffer** | Creates a zone of fixed or variable distance around a feature — the standard tool for proximity/catchment analysis. |
| **Clip** | Cuts an input layer down to only the portion that falls inside an overlay polygon, without altering attribute values. |
| **Convex Hull** | Wraps the smallest possible convex polygon around a set of features — useful for approximating an outer extent. |
| **Dissolve** | Merges adjacent polygons that share an attribute value (or a selection) into a single combined feature, erasing internal borders. |
| **Union** | Overlays two layers and produces a new layer containing every unique overlapping and non-overlapping portion from both, with combined attributes. |

Each of these was tested directly on the digitized Tamil Nadu layers — buffering a set of districts, clipping the state boundary against the buffer, dissolving a selected group of districts into one region, and unioning two overlapping layers to preserve both attribute sets.

---

## Takeaways

- **Georeferencing and rubber sheeting** solve the same problem — tying a raster to real-world coordinates — but rubber sheeting is the fallback when no coordinate grid exists on the source image.
- **Digitizing your own vector layers** (rather than only ever using pre-made shapefiles) is a core skill — the Split Features tool in particular saves significant time on shared borders.
- **Joining tabular data via a common key field** is what turns a plain boundary layer into an actual thematic map.
- **Selection and geoprocessing tools** (by expression, by location, buffer, clip, dissolve, union) are the actual analytical backbone of most everyday GIS work — far more so than any single "impressive" output map.

---

*All maps built in QGIS 3.28 using Census of India (2011) district-level demographic data for Tamil Nadu. Base administrative and road-network layers were digitized from a georeferenced scanned source map as part of this workflow.*
