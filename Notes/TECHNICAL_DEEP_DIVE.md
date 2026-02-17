# GIS Map Portal — Technical Deep Dive

This document provides **implementation-level details** for interview questions: how basemap switching preserves zoom/location, how layers are managed, how routes are generated, what components/entities exist, and the exact code flow.

---

## Table of Contents

1. [Basemap Switching — No Map Regeneration](#1-basemap-switching--no-map-regeneration)
2. [Layer Management — Dynamic Visibility & Opacity](#2-layer-management--dynamic-visibility--opacity)
3. [Route Generation — OSRM Integration](#3-route-generation--osrm-integration)
4. [Main Components — Frontend & Backend](#4-main-components--frontend--backend)
5. [Entities — Database Schema & JPA Mapping](#5-entities--database-schema--jpa-mapping)
6. [Implementation Details — Code Flow](#6-implementation-details--code-flow)

---

## 1. Basemap Switching — No Map Regeneration

### Problem
When a user switches basemaps (e.g., from OpenStreetMap to Satellite), we must **preserve the current view** (center, zoom level) and **not recreate the entire map** (which would reset everything and lose user context).

### Solution: Source Swapping Pattern

**Location:** `frontend/src/hooks/useMap.js` (lines 48-58)

```javascript
// Switch basemap; preserve center and zoom
useEffect(() => {
  if (!map || !basemapLayerRef.current) return;
  const view = map.getView();
  const center = view.getCenter();  // Save current center
  const zoom = view.getZoom();      // Save current zoom
  const newSource = getBasemapLayer(basemapId);
  basemapLayerRef.current.setSource(newSource);  // Swap source only
  if (center) view.setCenter(center);           // Restore center
  if (zoom != null) view.setZoom(zoom);         // Restore zoom
}, [map, basemapId]);
```

### How It Works

1. **Map Creation (Once):** When the component mounts, `useMap` creates a single `Map` instance with one `TileLayer` (basemap) and stores a ref to that layer:
   ```javascript
   const basemapLayer = new TileLayer({ source, zIndex: 0 });
   basemapLayerRef.current = basemapLayer;
   ```

2. **Source Swapping (On Change):** When `basemapId` changes (user picks a different basemap):
   - We **read** the current view state (`center`, `zoom`) from `map.getView()`.
   - We **create a new source** (`OSM()`, `XYZ(...)`, etc.) via `getBasemapLayer(basemapId)`.
   - We **swap the source** on the existing layer: `basemapLayerRef.current.setSource(newSource)`.
   - We **restore** the view state: `view.setCenter(center)` and `view.setZoom(zoom)`.

3. **Why This Works:**
   - OpenLayers `TileLayer.setSource()` replaces the source **without removing the layer** from the map.
   - The layer's zIndex, visibility, and position in the layer stack remain unchanged.
   - The view (center/zoom) is independent of layers; we explicitly preserve it.
   - **No map regeneration:** The `Map` instance, all other layers (WMS, places, distance), and event listeners remain intact.

### Basemap Source Factory

**Location:** `frontend/src/constants/basemaps.js`

```javascript
export function getBasemapLayer(id) {
  switch (id) {
    case BASEMAP_OSM:
      return new OSM();  // ol/source/OSM
    case BASEMAP_OSM_HUMANITARIAN:
      return new XYZ({
        url: 'https://{a-c}.tile.openstreetmap.fr/hot/{z}/{x}/{y}.png',
        attributions: '...',
      });
    case BASEMAP_ESRI_IMAGERY:
      return new XYZ({
        url: 'https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}',
      });
    // ... more options
  }
}
```

Each basemap returns a **source object** (not a layer). The layer wrapper is created once; only the source changes.

### Interview Answer

**Q: How do you switch basemaps without regenerating the map?**

**A:** We use OpenLayers' `layer.setSource()` method. The map and layer instances are created once. When the user selects a different basemap, we:
1. Save the current view center and zoom.
2. Create a new source (OSM, XYZ, etc.) based on the selected basemap ID.
3. Call `basemapLayerRef.current.setSource(newSource)` to swap the source.
4. Restore the saved center and zoom.

This preserves the user's view and doesn't recreate the map, so all other layers, event handlers, and state remain intact.

---

## 2. Layer Management — Dynamic Visibility & Opacity

### Problem
Users need to toggle WMS overlay layers on/off and adjust opacity (0-100%) **without recreating layers** or affecting other layers.

### Solution: State-Driven Layer Properties

**Location:** `frontend/src/hooks/useMap.js` (lines 60-70)

```javascript
// Sync WMS visibility and opacity from state
useEffect(() => {
  if (!wmsLayerStates) return;
  wmsLayersRef.current.forEach((layer, i) => {
    const state = wmsLayerStates[i];
    if (state) {
      layer.setVisible(state.visible);      // Show/hide
      layer.setOpacity(state.opacity);      // 0.0 to 1.0
    }
  });
}, [wmsLayerStates]);
```

### How It Works

1. **Layer Creation (Once):** On mount, `useMap` creates all WMS layers upfront:
   ```javascript
   const wmsLayers = WMS_LAYERS.map((def, i) =>
     createWmsImageLayer(def, 10 + i)
   );
   wmsLayersRef.current = wmsLayers;
   ```
   Each layer is a `TileLayer` with a `TileWMS` source. They're added to the map immediately but can be hidden.

2. **State Structure:** In `App.jsx`, we maintain an array of layer states:
   ```javascript
   const [wmsLayerStates, setWmsLayerStates] = useState(
     WMS_LAYERS.map(() => ({ visible: false, opacity: 0.7 }))
   );
   ```
   Each element corresponds to one WMS layer: `{ visible: boolean, opacity: number }`.

3. **State Updates:** When the user toggles a checkbox or moves the opacity slider:
   ```javascript
   const setWmsVisible = useCallback((index, visible) => {
     setWmsLayerStates((prev) => {
       const next = [...prev];
       next[index] = { ...next[index], visible };
       return next;
     });
   }, []);
   ```

4. **Layer Sync:** The `useEffect` in `useMap` watches `wmsLayerStates` and calls:
   - `layer.setVisible(state.visible)` — Controls whether tiles are rendered.
   - `layer.setOpacity(state.opacity)` — Controls transparency (0.0 = fully transparent, 1.0 = opaque).

5. **Why This Works:**
   - Layers exist in memory; we're only changing properties.
   - OpenLayers handles re-rendering when visibility/opacity changes.
   - Tiles are cached; toggling visibility doesn't re-fetch tiles.
   - Other layers (basemap, places, distance) are unaffected.

### WMS Layer Creation

**Location:** `frontend/src/constants/wmsLayers.js`

```javascript
export function createWmsImageLayer(def, zIndex) {
  const source = new TileWMS({
    url: def.url,
    params: { LAYERS: def.layers, VERSION: def.version },
    tileSize: 256,
    transition: 150,
  });
  const layer = new TileLayer({ source, zIndex, visible: true });
  layer.set('wmsDef', def);  // Store definition for GetFeatureInfo
  return layer;
}
```

**Why TileWMS:** Tiles load incrementally and are cached. When visibility changes, cached tiles are shown/hidden instantly.

### Interview Answer

**Q: How do you manage layer visibility and opacity?**

**A:** We maintain a React state array (`wmsLayerStates`) where each element holds `{ visible, opacity }` for one WMS layer. Layers are created once on mount and stored in a ref. When state changes (user toggles checkbox or slider), a `useEffect` syncs the state to layers by calling `layer.setVisible()` and `layer.setOpacity()`. This updates OpenLayers' internal state, which triggers re-rendering without recreating layers or re-fetching tiles.

---

## 3. Route Generation — OSRM Integration

### Problem
Users need to see a **road route** (not straight line) between two points, with distance and driving time.

### Solution: OSRM API + Vector Layer

**Location:** `frontend/src/utils/osrmRoute.js` and `frontend/src/components/MapView.jsx` (lines 444-490)

### Step-by-Step Flow

#### 1. User Clicks Two Points

When the user selects "Distance" tool and clicks the map twice:

```javascript
// In MapView.jsx - handleMapClick
if (toolMode === TOOL_DISTANCE && onDistancePoint) {
  onDistancePoint(lonLat[1], lonLat[0]);  // lat, lon
  return;
}
```

This calls `App.jsx`'s `handleDistancePoint`, which updates state:
```javascript
setDistancePoints((prev) => {
  if (prev.length >= 2) return prev;
  const next = [...prev, { lat, lon }];
  if (next.length === 2) {
    // Trigger API calls
    getDistance(...).then(setDistanceResult);
  }
  return next;
});
```

#### 2. Draw Point Markers

**Location:** `MapView.jsx` (lines 444-459)

```javascript
useEffect(() => {
  const source = vectorSourceRef.current;
  if (!source) return;
  source.clear();
  if (distancePoints.length === 0) return;

  const p1 = distancePoints[0];
  const f1 = new Feature({
    geometry: new Point(fromLonLat([p1.lon, p1.lat])),
    role: 'point1'
  });
  source.addFeature(f1);

  if (distancePoints.length === 1) return;

  const p2 = distancePoints[1];
  const f2 = new Feature({
    geometry: new Point(fromLonLat([p2.lon, p2.lat])),
    role: 'point2'
  });
  source.addFeature(f2);
  // ... route fetching below
}, [distancePoints]);
```

Two `Point` features are added to the distance layer's `VectorSource` with roles `'point1'` and `'point2'`. The style function renders them as blue circles with numbers "1" and "2".

#### 3. Fetch Road Route from OSRM

**Location:** `frontend/src/utils/osrmRoute.js`

```javascript
export async function getRoadRoute(lon1, lat1, lon2, lat2) {
  const coords = `${lon1},${lat1};${lon2},${lat2}`;
  const url = `https://router.project-osrm.org/route/v1/driving/${coords}?overview=full&geometries=geojson`;
  
  const res = await fetch(url);
  const data = await res.json();
  const route = data.routes?.[0];
  
  if (data.code !== 'Ok' || !route?.geometry?.coordinates?.length) {
    return null;
  }
  
  return {
    coordinates: route.geometry.coordinates,  // [[lon, lat], ...]
    distanceMeters: route.distance,
    durationSeconds: route.duration,
  };
}
```

**OSRM API Details:**
- Endpoint: `https://router.project-osrm.org/route/v1/driving/{coords}`
- Format: `{lon1},{lat1};{lon2},{lat2}`
- Parameters: `overview=full` (full geometry), `geometries=geojson` (GeoJSON coordinates)
- Response: `{ routes: [{ geometry: { coordinates: [[lon,lat],...] }, distance, duration }] }`

#### 4. Draw Route Line

**Location:** `MapView.jsx` (lines 461-489)

```javascript
let cancelled = false;
getRoadRoute(p1.lon, p1.lat, p2.lon, p2.lat)
  .then((result) => {
    if (cancelled || !vectorSourceRef.current) return;
    const coords = result?.coordinates;
    const pts = (coords && coords.length >= 2)
      ? coords.map(([lon, lat]) => fromLonLat([lon, lat]))  // Transform to EPSG:3857
      : [fromLonLat([p1.lon, p1.lat]), fromLonLat([p2.lon, p2.lat])];  // Fallback: straight line
    
    const line = new Feature({
      geometry: new LineString(pts),
      role: 'line'
    });
    vectorSourceRef.current.addFeature(line);
    
    if (onRouteResult && result && (result.distanceMeters > 0 || result.durationSeconds > 0)) {
      onRouteResult({ distanceMeters: result.distanceMeters, durationSeconds: result.durationSeconds });
    }
  })
  .catch(() => {
    // Fallback: draw straight line if OSRM fails
    const line = new Feature({
      geometry: new LineString([
        fromLonLat([p1.lon, p1.lat]),
        fromLonLat([p2.lon, p2.lat]),
      ]),
      role: 'line',
    });
    vectorSourceRef.current.addFeature(line);
  });
return () => { cancelled = true; };  // Cleanup on unmount
```

**Key Points:**
- OSRM returns coordinates in WGS84 (`[[lon, lat], ...]`).
- We transform them to EPSG:3857 (Web Mercator) using `fromLonLat()` for display.
- We create a `LineString` geometry with these points.
- The style function renders it as a white outline + blue center for visibility.
- If OSRM fails, we draw a straight line as fallback.

#### 5. Display Results

The route result (distance in meters, duration in seconds) is passed to `App.jsx` via `onRouteResult`, which updates state and displays it in the sidebar:

```javascript
{distanceResult != null && (
  <div>Geographical Distance: {distanceResult.distanceKm.toFixed(2)} km</div>
)}
{routeResult != null && (
  <div>
    Road Distance: {(routeResult.distanceMeters / 1000).toFixed(2)} km
    Time: {formatDuration(routeResult.durationSeconds)}
  </div>
)}
```

### Why OSRM?

- **Free public API** (no API key needed for demo).
- Returns **full geometry** (not just distance), so we can draw the route.
- Provides **driving distance and time** (not just straight-line).
- Uses OpenStreetMap data (consistent with our basemap).

### Interview Answer

**Q: How do you generate the route between two points?**

**A:** When the user clicks two points in Distance mode:
1. We draw two point markers immediately.
2. We call OSRM's public routing API (`router.project-osrm.org`) with the two coordinates.
3. OSRM returns a GeoJSON LineString with road geometry, distance in meters, and duration in seconds.
4. We transform the coordinates from WGS84 to EPSG:3857 and create an OpenLayers `LineString` feature.
5. We add it to a vector layer with a style (white outline + blue center) so it's visible.
6. We display both geographical distance (from our API) and road distance/time (from OSRM) in the sidebar.
7. If OSRM fails, we fall back to a straight line.

---

## 4. Main Components — Frontend & Backend

### Frontend Components

| Component | File | Responsibility |
|-----------|------|----------------|
| **App** | `frontend/src/App.jsx` | Root component; manages all state (basemap, WMS, tools, distance, places, search); coordinates Sidebar and MapView; handles API calls (createPlace, getDistance). |
| **MapView** | `frontend/src/components/MapView.jsx` | OpenLayers map lifecycle; manages all map layers (places, distance, search highlight); handles click/hover events; popup overlay; "My location" button. |
| **useMap** | `frontend/src/hooks/useMap.js` | Custom hook; creates Map instance once; manages basemap source swapping; syncs WMS visibility/opacity from state. |
| **Sidebar** | `frontend/src/components/Sidebar.jsx` | Collapsible panel UI; renders basemap list, WMS checkboxes/sliders, tool buttons; no business logic. |
| **SidebarSection** | `frontend/src/components/Sidebar.jsx` | Reusable section wrapper (title, icon, children). |
| **AppHeader** | `frontend/src/components/AppHeader.jsx` | Header with logo/title; mobile menu button. |
| **LocationSearch** | `frontend/src/components/LocationSearch.jsx` | Search input; calls `geocode()` (Nominatim); calls `onSelect(result)` on success. |
| **AddLocationModal** | `frontend/src/components/AddLocationModal.jsx` | Modal for adding a place; name/type inputs; calls `onSubmit(name, type)`; handles loading/success/error. |

### Frontend Utilities

| Utility | File | Purpose |
|---------|------|---------|
| **placesApi** | `frontend/src/api/placesApi.js` | API client; `createPlace()`, `getPlaces()`, `getNearby()`, `getNearest()`, `getDistance()`; builds URLs from config; handles fetch errors. |
| **geocode** | `frontend/src/utils/geocode.js` | Nominatim geocoding; searches place names; returns `{ lat, lon, displayName, bbox }`. |
| **osrmRoute** | `frontend/src/utils/osrmRoute.js` | OSRM routing API; returns road geometry, distance, duration. |
| **wmsGetFeatureInfo** | `frontend/src/utils/wmsGetFeatureInfo.js` | Builds WMS GetFeatureInfo URL (extent, pixel, size, CRS, BBOX). |
| **config** | `frontend/src/config.js` | Centralized config; `apiBase` from `VITE_API_BASE` env var. |
| **basemaps** | `frontend/src/constants/basemaps.js` | Basemap definitions; `getBasemapLayer(id)` factory. |
| **wmsLayers** | `frontend/src/constants/wmsLayers.js` | WMS layer definitions; `createWmsImageLayer(def, zIndex)` factory. |

### Backend Components

| Component | Package | Responsibility |
|-----------|---------|----------------|
| **PlaceController** | `com.gis.places.controller` | REST controller; `/api/places` endpoints; delegates to PlaceService; returns ResponseEntity. |
| **PlaceService** | `com.gis.places.service` | Interface; contract for place operations. |
| **PlaceServiceImpl** | `com.gis.places.service.impl` | Service implementation; validation; transaction management; calls repository; maps to GeoJSON. |
| **PlaceRepository** | `com.gis.places.repository` | JPA repository; extends `JpaRepository<Place, Long>`; custom methods (`findNearbyActive`, `findNearestActive`); extends `PlaceRepositoryCustom`. |
| **PlaceRepositoryCustom** | `com.gis.places.repository` | Interface; `calculateDistanceMeters(lat1, lon1, lat2, lon2)`. |
| **PlaceRepositoryImpl** | `com.gis.places.repository.impl` | Custom repository implementation; native SQL for `ST_Distance` calculation. |
| **GeoJsonMapper** | `com.gis.places.util` | Maps `Place` → GeoJSON Feature; `List<Place>` → FeatureCollection. |
| **CoordinateValidator** | `com.gis.places.util` | Validates lat/lon ranges; validates radius; throws `InvalidRequestException`. |
| **GlobalExceptionHandler** | `com.gis.places.exception` | `@RestControllerAdvice`; maps exceptions to HTTP status + ErrorResponse. |
| **SecurityConfig** | `com.gis.places.config` | Spring Security; CORS configuration; endpoint permissions. |
| **AppProperties** | `com.gis.places.config` | Configuration properties (CORS origins, default radius, max radius). |

### Backend DTOs

| DTO | Package | Purpose |
|-----|---------|---------|
| **CreatePlaceRequest** | `com.gis.places.dto` | Request body for POST /api/places; Bean Validation annotations. |
| **DistanceResponse** | `com.gis.places.dto` | Response for GET /api/places/distance; `distanceKm`, `distanceMeters`. |
| **ErrorResponse** | `com.gis.places.dto` | Standard error response; `status`, `code`, `message`, `path`, optional `field`/`fieldErrors`. |

### Backend Exceptions

| Exception | Package | When Thrown |
|-----------|---------|------------|
| **InvalidRequestException** | `com.gis.places.exception` | Invalid coordinates, radius out of range; 400 Bad Request. |
| **ResourceNotFoundException** | `com.gis.places.exception` | No nearest place found; 404 Not Found. |

---

## 5. Entities — Database Schema & JPA Mapping

### BaseEntity (Abstract)

**Location:** `backend/src/main/java/com/gis/places/entity/BaseEntity.java`

**Purpose:** Common fields for all entities (audit trail, soft delete).

**Fields:**
- `uuid` (UUID, unique, not null, not updatable) — Generated on `@PrePersist`.
- `createdBy` (String, not null) — Set by service (e.g., "system").
- `createdDate` (OffsetDateTime, not null) — Set on `@PrePersist`.
- `modifiedBy` (String, nullable) — Set on update.
- `modifiedDate` (OffsetDateTime, nullable) — Set on `@PreUpdate`.
- `active` (Boolean, not null, default true) — Soft delete flag.

**JPA Annotations:**
- `@MappedSuperclass` — Not a table itself; fields inherited by child entities.
- `@PrePersist` — `onCreate()` sets `uuid` and `createdDate`.
- `@PreUpdate` — `onUpdate()` sets `modifiedDate`.

**Why:** Consistent audit trail and soft delete across entities without duplicating code.

### Place Entity

**Location:** `backend/src/main/java/com/gis/places/entity/Place.java`

**Purpose:** Represents a place (point of interest) with name, type, and geographic location.

**Inheritance:** Extends `BaseEntity` (gets `uuid`, audit fields, `active`).

**Fields:**
- `id` (Long, primary key, auto-generated) — Database ID.
- `name` (String, not null, max 255) — Place name.
- `type` (String, not null, max 100) — Place type (e.g., "park", "restaurant").
- `location` (JTS `Point`, not null) — Geographic point stored as PostGIS `geography(Point,4326)`.

**JPA Annotations:**
- `@Entity` — JPA entity.
- `@Table(name = "place")` — Table name.
- `@Id` + `@GeneratedValue(strategy = GenerationType.IDENTITY)` — Auto-increment primary key.
- `@Column(name = "location", columnDefinition = "geography(Point,4326)")` — PostGIS geography type.
- `@JdbcType(PGCastingGeographyJdbcType.class)` — Hibernate Spatial type mapping.

**Database Schema (Liquibase):**

```sql
CREATE TABLE place (
    id BIGSERIAL PRIMARY KEY,
    uuid UUID NOT NULL UNIQUE,
    created_by VARCHAR(255) NOT NULL,
    created_date TIMESTAMP NOT NULL,
    modified_by VARCHAR(255),
    modified_date TIMESTAMP,
    active BOOLEAN NOT NULL DEFAULT true,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(100) NOT NULL,
    location GEOGRAPHY(POINT, 4326) NOT NULL
);

CREATE INDEX idx_place_location_gist ON place USING GIST (location);
CREATE INDEX idx_place_active ON place (active);
```

**Key Points:**
- `location` is `geography(Point,4326)` — Uses ellipsoid for distance calculations.
- GIST index on `location` — Efficient for spatial queries (`ST_DWithin`, `<->`).
- Index on `active` — Fast filtering for active places.

**Why Geography vs Geometry:**
- `geography` computes distances in meters on the ellipsoid (real-world distance).
- `geometry` in 4326 would compute distances in degrees (not useful for "within 5 km").
- `ST_Distance` on geography returns meters; on geometry it returns degrees.

### Interview Answer

**Q: What entities do you have, and how are they mapped?**

**A:** We have two entity classes:
1. **BaseEntity** — Abstract `@MappedSuperclass` with `uuid`, `createdBy`, `createdDate`, `modifiedBy`, `modifiedDate`, `active`. Uses `@PrePersist` and `@PreUpdate` to set timestamps and UUID.
2. **Place** — Extends `BaseEntity`. Fields: `id` (primary key), `name`, `type`, `location` (JTS Point). The `location` field is mapped to PostGIS `geography(Point,4326)` using `@JdbcType(PGCastingGeographyJdbcType.class)`. We use `geography` (not `geometry`) because it computes distances in meters on the ellipsoid, which is what we need for "within 5 km" queries. The database has a GIST index on `location` for efficient spatial queries.

---

## 6. Implementation Details — Code Flow

### Flow: Adding a Place

1. **User Action:** User selects "Add" tool, clicks map, fills modal, submits.
2. **Frontend (App.jsx):** `handleAddSubmit` calls `createPlace({ name, type, lat, lon })`.
3. **API Client (placesApi.js):** Builds URL, sends `POST /api/places` with JSON body.
4. **Backend (PlaceController):** Receives `@Valid @RequestBody CreatePlaceRequest`.
5. **Validation:** Bean Validation checks `@NotBlank`, `@Size`, `@DecimalMin`/`@DecimalMax`.
6. **Service (PlaceServiceImpl):** `CoordinateValidator.validateLatLon()`, creates JTS `Point` (lon, lat), sets name/type, saves via `placeRepository.save()`.
7. **Repository:** JPA/Hibernate persists `Place`; `@PrePersist` sets UUID and `createdDate`.
8. **Database:** INSERT into `place` table; PostGIS stores point as `geography(Point,4326)`.
9. **Response:** `GeoJsonMapper.toFeature()` converts `Place` → GeoJSON Feature; returns 201 Created.
10. **Frontend:** On success, closes modal, increments `placesRefreshTrigger`.
11. **MapView:** Effect watches `placesRefreshTrigger`, calls `getPlaces()`, updates `placesSourceRef.current` with new features; map redraws markers.

### Flow: Finding Nearby Places

1. **Query:** `GET /api/places/nearby?lat=40.78&lon=-73.97&radiusKm=5`
2. **Controller:** Extracts params, calls `placeService.findNearby(lat, lon, radiusKm)`.
3. **Service:** Validates coordinates and radius, converts `radiusKm` to meters, calls `placeRepository.findNearbyActive(lat, lon, radiusMeters)`.
4. **Repository:** Executes native query:
   ```sql
   SELECT p.* FROM place p
   WHERE p.active = true
   AND ST_DWithin(
       p.location,
       ST_SetSRID(ST_MakePoint(:lon, :lat), 4326)::geography,
       :radiusMeters
   )
   ORDER BY ST_Distance(p.location, ...)
   ```
5. **PostGIS:** `ST_DWithin` uses GIST index; `ST_Distance` orders results.
6. **Response:** `GeoJsonMapper.toFeatureCollection()` → GeoJSON FeatureCollection; returns 200 OK.

### Flow: Distance Calculation

1. **Query:** `GET /api/places/distance?lat1=40.78&lon1=-73.97&lat2=40.75&lon2=-73.99`
2. **Controller:** Extracts params, calls `placeService.getDistance(lat1, lon1, lat2, lon2)`.
3. **Service:** Validates both points, calls `placeRepository.calculateDistanceMeters(lat1, lon1, lat2, lon2)`.
4. **Custom Repository:** Executes native query:
   ```sql
   SELECT ST_Distance(
       ST_SetSRID(ST_MakePoint(:lon1, :lat1), 4326)::geography,
       ST_SetSRID(ST_MakePoint(:lon2, :lat2), 4326)::geography
   )
   ```
5. **PostGIS:** Returns distance in meters (ellipsoid calculation).
6. **Response:** `DistanceResponse(km)` → JSON `{ distanceKm, distanceMeters }`; returns 200 OK.

### Flow: WMS GetFeatureInfo

1. **User Action:** User selects "Info" tool, clicks map (with WMS layer visible).
2. **MapView:** `handleMapClick` gets click coordinate and pixel.
3. **Pixel Calculation:** `map.getPixelFromCoordinate(coord)` → `[i, j]`; `j = height - pixel[1]` (image coords).
4. **Extent/Size:** `map.getView().calculateExtent(size)` → current map extent in EPSG:3857.
5. **URL Building:** `getFeatureInfoUrl(def, { extent, width, height, i, j })` builds WMS GetFeatureInfo URL:
   ```
   ?REQUEST=GetFeatureInfo&SERVICE=WMS&VERSION=1.3.0&LAYERS=TOPO-WMS&
   QUERY_LAYERS=TOPO-WMS&INFO_FORMAT=application/vnd.ogc.gml&
   I=123&J=456&WIDTH=800&HEIGHT=600&CRS=EPSG:3857&BBOX=...
   ```
6. **Request:** `fetch(urlGml)` → WMS server returns GML or HTML.
7. **Parsing:** If GML, parse XML, extract `<Feature><Property>` elements, build HTML table.
8. **Display:** Set popup content, position overlay at click coordinate.

### Interview Answer

**Q: Walk me through how adding a place works end-to-end.**

**A:** 
1. User clicks map in Add mode → `MapView` converts coordinate to lat/lon → calls `onAddLocationClick(lat, lon)`.
2. `App` opens `AddLocationModal` with those coordinates.
3. User enters name/type and submits → `App` calls `createPlace({ name, type, lat, lon })`.
4. `placesApi.js` sends `POST /api/places` with JSON body.
5. `PlaceController` receives it; Bean Validation checks fields.
6. `PlaceServiceImpl.create()` validates coordinates, creates JTS `Point` (lon, lat), sets name/type, calls `placeRepository.save()`.
7. Hibernate persists `Place`; `@PrePersist` sets UUID and timestamp.
8. Database stores point as `geography(Point,4326)`.
9. Service maps `Place` → GeoJSON Feature via `GeoJsonMapper`.
10. Controller returns 201 Created with Feature.
11. Frontend closes modal, increments `placesRefreshTrigger`.
12. `MapView` effect watches trigger, calls `getPlaces()`, updates vector source; map redraws with new marker.

---

This document covers the **implementation mechanics** you'll need for technical interviews. Use it alongside `PROJECT_GUIDE.md` for complete coverage.
