# GIS Map Portal — Complete Technical Guide

This document explains **every component, endpoint, map layer, and design decision** in the project so you can answer interview questions confidently: **what** we built, **how** it works, **why** we chose it, and **when** things run.

---

## Table of Contents

1. [High-Level Architecture](#1-high-level-architecture)
2. [Backend — Components & Flow](#2-backend--components--flow)
3. [API Endpoints — Request/Response & Flow](#3-api-endpoints--requestresponse--flow)
4. [Map & OpenLayers — How Layers Work](#4-map--openlayers--how-layers-work)
5. [Frontend Components — Responsibility & Data Flow](#5-frontend-components--responsibility--data-flow)
6. [Interview Q&A — When, How, What, Why](#6-interview-qa--when-how-what-why)

---

## 1. High-Level Architecture

### What the application does

- **Backend:** REST API for **places** (points with name, type, lat/lon). Supports create, list all, find nearby (radius), find nearest, and straight-line distance between two points. All spatial logic uses **PostGIS** (PostgreSQL extension).
- **Frontend:** Single-page map app. Users can switch **basemaps**, toggle **WMS overlay layers**, use **tools** (Add place, Distance, WMS Info), **search locations** (geocoding), view **places** as red markers, measure **road distance** (OSRM), and see **popups** for places and WMS feature info.

### Tech choices (why)

| Choice | Why |
|--------|-----|
| **Spring Boot 3 + Java 21** | Fast development, embedded server, strong ecosystem; Java 21 for LTS and modern APIs. |
| **PostgreSQL + PostGIS** | Relational DB with spatial types (`geography(Point,4326)`), indexes (GIST), and functions (`ST_DWithin`, `ST_Distance`, `<->` for nearest). |
| **Liquibase** | Versioned schema (PostGIS extension, `place` table, indexes); no `ddl-auto: create` in production. |
| **React + Vite** | Component-based UI; Vite for fast HMR and simple proxy to backend. |
| **OpenLayers** | Mature, open-source web mapping; supports XYZ tiles, WMS, vector layers, overlays, and projections (EPSG:3857 for display, 4326 for data). |

### Request flow (end-to-end)

1. User action in browser (e.g. click “Add” then click map) → React state updates.
2. Frontend calls backend (e.g. `POST /api/places`) via `fetch` from `placesApi.js`.
3. Backend: `PlaceController` → `PlaceService` → `PlaceRepository` + `GeoJsonMapper` → JSON response.
4. Frontend: on success, state updates (e.g. close modal, increment `placesRefreshTrigger`).
5. Map: `MapView` refetches `GET /api/places` and redraws places layer (OpenLayers `VectorSource` + GeoJSON).

---

## 2. Backend — Components & Flow

### Package structure

```
com.gis.places
├── config          # Security, CORS, app properties (radius, CORS origins)
├── constants       # ApiConstants (paths, params), ErrorCodes
├── controller      # PlaceController (REST)
├── dto             # CreatePlaceRequest, DistanceResponse, ErrorResponse
├── entity          # Place, BaseEntity (uuid, audit, active)
├── exception       # InvalidRequestException, ResourceNotFoundException, GlobalExceptionHandler
├── repository      # PlaceRepository (JPA + custom), PlaceRepositoryCustom, PlaceRepositoryImpl
├── service         # PlaceService (interface), PlaceServiceImpl
└── util            # GeoJsonMapper, CoordinateValidator
```

### Place entity

- **What:** JPA entity mapping to table `place`. Fields: `id`, `uuid`, `name`, `type`, `location` (JTS `Point`), plus `BaseEntity` fields (`createdBy`, `createdDate`, `modifiedBy`, `modifiedDate`, `active`).
- **How:** `location` is `geography(Point,4326)` in PostGIS. Hibernate maps it via `@JdbcType(PGCastingGeographyJdbcType.class)` (Hibernate Spatial / PostGIS dialect). Coordinates stored as (longitude, latitude) in WGS84.
- **Why geography:** PostGIS `geography` type computes distances in meters on the ellipsoid; `geometry` would be in SRID units. We need “radius in km” and “nearest place” in real-world distance.

### BaseEntity

- **What:** `@MappedSuperclass` with `uuid`, `createdBy`, `createdDate`, `modifiedBy`, `modifiedDate`, `active`.
- **How:** `@PrePersist` sets `uuid` and `createdDate`; `@PreUpdate` sets `modifiedDate`. All entities (e.g. `Place`) extend it.
- **Why:** Consistent audit and soft-delete (active flag) across entities.

### PlaceRepository

- **What:** `JpaRepository<Place, Long>` plus custom methods: `findAllByActiveTrue`, `findNearbyActive`, `findNearestActive`, and (via `PlaceRepositoryCustom`) `calculateDistanceMeters`.
- **How:**
  - **Nearby:** Native query with `ST_DWithin(place.location, ST_SetSRID(ST_MakePoint(:lon,:lat),4326)::geography, :radiusMeters)` and `ORDER BY ST_Distance(...)`.
  - **Nearest:** `ORDER BY place.location <-> ST_SetSRID(ST_MakePoint(:lon,:lat),4326)::geography LIMIT 1` (PostGIS KNN operator `<->`).
  - **Distance:** Custom implementation in `PlaceRepositoryImpl`: single native query `SELECT ST_Distance( geography1, geography2 )` with two `ST_SetSRID(ST_MakePoint(...),4326)::geography` points.
- **Why native for spatial:** PostGIS functions (`ST_DWithin`, `<->`, `ST_Distance`) are easiest and most efficient in raw SQL; JPA criteria don’t standardize these.

### PlaceRepositoryCustom + PlaceRepositoryImpl

- **What:** Custom interface `calculateDistanceMeters(lat1, lon1, lat2, lon2)`; implementation uses `EntityManager.createNativeQuery(DISTANCE_SQL)`.
- **Why:** JPA doesn’t support “return a single double from a spatial function”; custom repository keeps the rest of the app in Java/Spring style while using one native query.

### PlaceService / PlaceServiceImpl

- **What:** Single service implementing create, findAll, findNearby, findNearest, getDistance. Validates coordinates and radius, converts entities to GeoJSON, and throws domain exceptions (e.g. `ResourceNotFoundException` for no nearest place).
- **How:**
  - **Create:** `CoordinateValidator.validateLatLon` → build JTS `Point` (lon, lat) → `placeRepository.save` → `GeoJsonMapper.toFeature`.
  - **FindAll:** `findAllByActiveTrue()` → `toFeatureCollection`.
  - **FindNearby:** validate lat/lon and radius (km) → convert to meters → `findNearbyActive` → `toFeatureCollection`.
  - **FindNearest:** validate → `findNearestActive` → `toFeature` or throw `ResourceNotFoundException(NO_PLACE_NEARBY)`.
  - **Distance:** validate both points → `calculateDistanceMeters` → `DistanceResponse(km)`.
- **Why service layer:** Keeps controller thin; centralizes validation, transaction boundaries (`@Transactional`), and mapping.

### GeoJsonMapper

- **What:** Maps `Place` → GeoJSON Feature (type, id, geometry, properties) and `List<Place>` → FeatureCollection. Geometry is `Point` with `[lon, lat]` (RFC 7946).
- **Why GeoJSON:** Standard for web maps; OpenLayers can read it directly (`ol/format/GeoJSON`) with `dataProjection: 'EPSG:4326'`, `featureProjection: 'EPSG:3857'`.

### CoordinateValidator

- **What:** Validates lat ∈ [-90, 90], lon ∈ [-180, 180], radiusKm ∈ (0, maxRadiusKm]. Throws `InvalidRequestException` with error code and optional field name.
- **Why:** Single place for validation rules; used by service before calling repository; consistent error codes for the API.

### CreatePlaceRequest (DTO)

- **What:** Request body for `POST /api/places`: `name`, `type`, `lat`, `lon` with Bean Validation (`@NotBlank`, `@Size`, `@NotNull`, `@DecimalMin`/`@DecimalMax`).
- **How:** Controller uses `@Valid @RequestBody CreatePlaceRequest`; validation runs before the method; failures handled by `GlobalExceptionHandler` → 400 with `fieldErrors`.

### DistanceResponse (DTO)

- **What:** JSON: `distanceKm`, `distanceMeters` (rounded). Built from `DistanceResponse(km)` in service.
- **Why:** Explicit contract for the frontend; `distanceMeters` derived once so clients don’t have to recalculate.

### PlaceController

- **What:** REST controller; base path `/api/places`; produces JSON. Endpoints: POST (create), GET (findAll), GET /nearby, GET /nearest, GET /distance.
- **How:** Delegates to `PlaceService`; returns `ResponseEntity` with status (201 for create, 200 for reads). Query params and body validated by Spring and validators.
- **Why single controller:** All place-related operations in one place; API constants in `ApiConstants` to avoid typos.

### GlobalExceptionHandler

- **What:** `@RestControllerAdvice` that maps exceptions to HTTP status and `ErrorResponse` (status, error, code, message, path, optional field/fieldErrors).
- **Handled:** `MethodArgumentNotValidException` → 400 + fieldErrors; `InvalidRequestException` → 400 + code/message/field; `ResourceNotFoundException` → 404; `MissingServletRequestParameterException` → 400; `AccessDeniedException` → 403; `Exception` → 500 with generic message (no stack trace to client).
- **Why:** Consistent API error shape; safe messages in production; logging in handler for 500.

### SecurityConfig

- **What:** Two filter chains. (1) For `/api/**`: permit only listed paths (`/api/places`, `/api/places/nearby`, etc.), no auth, CSRF disabled, stateless, CORS from `AppProperties`. (2) Default: permit all (e.g. health, static).
- **How:** CORS bean allows origins from config (e.g. localhost:5173, localhost:3000); methods GET/POST/PUT/DELETE/OPTIONS; credentials true.
- **Why:** Frontend on different port/origin must be allowed; no auth in this project scope, so no JWT/session.

### Database (Liquibase)

- **001-enable-postgis:** `CREATE EXTENSION IF NOT EXISTS postgis;`
- **002-create-place-table:** `place` table with id, uuid, audit columns, active, name, type, `location GEOGRAPHY(POINT, 4326) NOT NULL`.
- **002-place-location-gist-index:** `CREATE INDEX idx_place_location_gist ON place USING GIST (location);` for spatial queries.
- **002-place-active-index:** `CREATE INDEX idx_place_active ON place (active);` for filtered queries.

**Why Liquibase:** Reproducible schema; safe for production (no ddl-auto); PostGIS and GIST index are required for performance.

---

## 3. API Endpoints — Request/Response & Flow

### POST /api/places — Create place

- **When:** User clicks map in “Add” mode, fills name/type in modal, submits.
- **Request:** JSON body: `name`, `type`, `lat`, `lon`. Validated by Bean Validation and `CoordinateValidator` in service.
- **Flow:** Controller → PlaceService.create → validate → new Place + JTS Point → save → GeoJsonMapper.toFeature → 201 + Feature.
- **Response:** GeoJSON Feature (id, geometry Point, properties with id, uuid, name, type, active).
- **Errors:** 400 (validation/coordinates), 500 (generic message).

### GET /api/places — List all active places

- **When:** Map loads and whenever places are refreshed (e.g. after adding one); used to draw red markers.
- **Request:** No body; no required params.
- **Flow:** Controller → PlaceService.findAll → findAllByActiveTrue → toFeatureCollection → 200.
- **Response:** GeoJSON FeatureCollection of Features (same structure as single feature).
- **Why no pagination:** Small dataset for demo; for production you’d add limit/offset or cursor.

### GET /api/places/nearby — Places within radius

- **When:** Could be used by a “show nearby” feature; frontend currently uses GET /api/places for the main map.
- **Request:** Query params: `lat`, `lon`, optional `radiusKm` (default from app config, max 100).
- **Flow:** Controller → PlaceService.findNearby → validate lat/lon and radius → findNearbyActive(radiusMeters) → toFeatureCollection → 200.
- **Response:** GeoJSON FeatureCollection (possibly empty).
- **Errors:** 400 (invalid lat/lon/radius), 500.

### GET /api/places/nearest — Single nearest place

- **When:** E.g. “nearest place to my location”; frontend could call it when user clicks “nearest.”
- **Request:** Query params: `lat`, `lon`.
- **Flow:** Controller → PlaceService.findNearest → findNearestActive → toFeature or throw ResourceNotFoundException → 200 or 404.
- **Response:** 200 = single GeoJSON Feature; 404 = body with code `NO_PLACE_NEARBY`, message.
- **Errors:** 400 (invalid coords), 404 (no places), 500.

### GET /api/places/distance — Straight-line distance

- **When:** User selects two points in “Distance” tool; frontend calls this for geographical distance and also fetches road route from OSRM.
- **Request:** Query params: `lat1`, `lon1`, `lat2`, `lon2`.
- **Flow:** Controller → PlaceService.getDistance → validate both points → calculateDistanceMeters (PostGIS ST_Distance) → DistanceResponse → 200.
- **Response:** `{ "distanceKm": number, "distanceMeters": number }`.
- **Errors:** 400 (invalid coords), 500.

---

## 4. Map & OpenLayers — How Layers Work

### How the map is created (useMap hook)

- **When:** Once when the `MapView` div is mounted; `mapRef` is the div element.
- **What:** `useMap(mapRef, basemapId, wmsLayerStates)` creates one `Map` instance with:
  - **View:** `ol/View` with center (0, 20 in 3857), zoom 2, projection EPSG:3857.
  - **Layers:** One basemap TileLayer (from `getBasemapLayer(basemapId)`) + one TileLayer per WMS definition (from `createWmsImageLayer(def, zIndex)`).
- **Returns:** `{ map, wmsLayersRef, basemapLayerRef }`. Map is stored in state; refs point to the layer instances so we can change source (basemap) or visibility/opacity (WMS) without recreating the map.
- **Cleanup:** On unmount, `map.setTarget(undefined)` so OpenLayers releases the DOM.

### Layer stack (bottom to top)

1. **Basemap (zIndex 0)** — Single TileLayer; source is swapped when user picks another basemap (OSM, Humanitarian OSM, Esri Imagery, Esri Streets). **Why TileLayer:** Pre-rendered tiles (XYZ/OSM); fast pan/zoom and caching.
2. **WMS overlays (zIndex 10, 11, …)** — One TileLayer per WMS definition; each source is TileWMS (not ImageWMS). **Why TileWMS:** Tiles load incrementally and are cached; better UX than one big ImageWMS per view change.
3. **Places layer (zIndex 40)** — VectorLayer with VectorSource; features from GeoJSON (GET /api/places). Style: red circle. **Why vector:** Points are dynamic (from our API); we need to refresh and re-style without new tile URLs.
4. **Distance layer (zIndex 50)** — VectorLayer: two point features (numbered 1 and 2) and optional LineString (road route from OSRM). Style function: blue circles with “1”/“2” for points; white+blue stroke for line. **Why separate layer:** Clear separation; easy to clear when user leaves Distance tool.
5. **Search highlight (zIndex 50)** — VectorLayer with polygon(s): dim overlay (world with hole) + outline of searched bbox. **Why:** Visually emphasize the searched area; click map clears it.

Popup is an `ol/Overlay`, not a layer; it’s attached to the map and positioned in pixel coordinates.

### Basemaps (constants/basemaps.js)

- **What:** List of `{ id, name }` and `getBasemapLayer(id)` returning an OpenLayers source.
- **Options:** OSM (ol/source/OSM), Humanitarian OSM (XYZ with HOT tiles), Esri World Imagery (XYZ), Esri World Street (XYZ). All are tile-based.
- **How layer is switched:** In useMap, when `basemapId` changes we do `basemapLayerRef.current.setSource(newSource)` and preserve view center/zoom so the map doesn’t jump.
- **Why multiple basemaps:** Users can choose context (street, satellite, humanitarian) without changing app code.

### WMS layers (constants/wmsLayers.js)

- **What:** Array of definitions: `url`, `layers`, `version`, `extent`, `title`, `description`. Two layers: Topography (TOPO-WMS) and OSM overlay (TOPO-OSM-WMS) from Mundialis.
- **How created:** `createWmsImageLayer(def, zIndex)` builds a TileWMS source (params: LAYERS, VERSION; tile size 256) and a TileLayer; we store `def` on the layer as `wmsDef` for GetFeatureInfo.
- **Visibility/opacity:** Controlled by React state `wmsLayerStates` (per-layer `visible`, `opacity`). useMap effect runs when `wmsLayerStates` changes and calls `layer.setVisible(state.visible)` and `layer.setOpacity(state.opacity)`.
- **Why TileWMS:** Same as above: tiled requests, caching, smoother interaction than ImageWMS.

### GetFeatureInfo (WMS Info tool)

- **When:** User selects “Info” tool and clicks the map (with at least one WMS overlay visible).
- **How:** MapView’s click handler gets click coordinate and pixel; for each visible WMS (top-down), we build a GetFeatureInfo URL via `getFeatureInfoUrl(def, { extent, width, height, i, j })`. We use current map extent and size (in 3857); for WMS 1.3.0 we pass CRS=EPSG:3857 and BBOX in 3857. Pixel (i, j) is from OpenLayers (i = pixel[0], j = height - pixel[1] for image coords). We try GML first; if no features, we try HTML. Response is parsed (GML: parse XML and build table; HTML: strip server error divs) and shown in the popup.
- **Why GML first:** Structured; we can show a clean table. HTML fallback for servers that don’t support GML.
- **Why top-down:** Topmost visible WMS is tried first so the user sees the top layer’s info when multiple overlap.

### Places on the map

- **When:** On map ready and whenever `placesRefreshTrigger` changes (e.g. after adding a place).
- **How:** MapView effect calls `getPlaces()` → GeoJSON FeatureCollection. We use `ol/format/GeoJSON` with `dataProjection: 'EPSG:4326'`, `featureProjection: 'EPSG:3857'` to read features, then `placesSourceRef.current.clear()` and `addFeatures(features)`. Style: red circle (RED_MARKER_STYLE).
- **Click/hover:** On singleclick we check `forEachFeatureAtPixel` with layerFilter `name === 'places'`; if hit, we show popup with name/type from feature properties. On pointermove we do the same for hover (show popup) and change cursor to pointer when over a place.

### Distance tool (two points + road route)

- **When:** User selects “Distance” and clicks two points. First click: store point 1; second: store point 2, then call backend `getDistance(lat1, lon1, lat2, lon2)` and frontend `getRoadRoute(lon1, lat1, lon2, lat2)` (OSRM).
- **How:** App state holds `distancePoints` (array of { lat, lon }). MapView receives them and draws two Point features and, when OSRM returns, a LineString along the road. Geographical distance comes from our API; road distance/duration from OSRM. Sidebar shows both; “Clear” resets points and results.
- **Why OSRM:** Free routing engine; returns geometry (for the line) and distance/duration so we can show “driving” vs “as the crow flies.”

### Location search (geocoding)

- **When:** User types in the search box and submits. `LocationSearch` calls `geocode(query)` (Nominatim).
- **How:** `geocode.js` requests Nominatim with User-Agent; parses first result; returns `{ lat, lon, displayName, bbox }`. Nominatim bbox is [south, north, west, east]; we convert to [west, south, east, north] for OpenLayers. App calls `goToLocation(result)`: if bbox exists, we set `searchHighlightBbox` and `view.fit(extent)`; else we animate center/zoom. MapView draws the search-highlight layer from `searchHighlightBbox` (dim + outline). Clicking the map clears the highlight.
- **Why Nominatim:** Free, OSM-based; returns bbox so we can fit and highlight. We set User-Agent to comply with usage policy.

### My location

- **When:** On map load we call `navigator.geolocation.getCurrentPosition`; there is also a “My location” button.
- **How:** On success we store center/zoom in a ref and animate the view. Button reuses stored location or requests again. No marker is drawn for “my location” in the current code; only view moves.
- **Why:** Lets users quickly return to their area after panning/zooming.

---

## 5. Frontend Components — Responsibility & Data Flow

### App.jsx

- **Responsibility:** Root state and layout. Holds: sidebar open/collapsed, basemap id, WMS layer states, tool mode, distance points/result/route, add modal, error, placesRefreshTrigger, searchHighlightBbox. Passes handlers and state to Sidebar and MapView; renders AppHeader, Sidebar, map area, distance toolbar, and AddLocationModal.
- **Data flow:** User picks basemap → setBasemapId → MapView (useMap) swaps basemap source. User toggles WMS → setWmsVisible/setWmsOpacity → wmsLayerStates → useMap syncs visibility/opacity. User selects tool → setToolMode → MapView uses it in click handler. Add place: MapView calls onAddLocationClick → setAddModal → user submits in modal → createPlace API → setPlacesRefreshTrigger → MapView refetches places. Distance: MapView calls onDistancePoint → App updates distancePoints and calls getDistance; MapView fetches OSRM and calls onRouteResult. Search: LocationSearch calls onSelect(result) → goToLocation → setSearchHighlightBbox + fit/extent; MapView draws highlight; map click → onClearSearchHighlight.

### MapView.jsx

- **Responsibility:** OpenLayers map lifecycle, all map layers (basemap + WMS from useMap; places, distance, search highlight added in MapView), click/hover handling for tools and place popups, popup overlay, “My location” button.
- **Props:** basemapId, wmsLayerStates, toolMode, onAddLocationClick, onDistancePoint, onClearDistance, onRouteResult, distancePoints, distanceResult, onMapReady, placesRefreshTrigger, searchHighlightBbox, onClearSearchHighlight.
- **Refs:** mapRef (div), popupRef (overlay + content div), vectorSourceRef (distance), placesSourceRef (places), wmsLayersRef (from useMap), userLocationRef, searchHighlightLayerRef.
- **Effects:** Create map (useMap); add popup overlay; onMapReady(map); geolocation on load; search highlight layer from searchHighlightBbox; singleclick → handleMapClick (tool dispatch or place popup); pointermove → cursor + place hover popup; distance layer update when distancePoints/OSRM change; places layer load when map + placesRefreshTrigger; clear distance layer when tool !== Distance.

### useMap.js

- **Responsibility:** Create Map once, set basemap source from basemapId, sync WMS visibility/opacity from wmsLayerStates. No creation of places/distance/highlight layers (those are in MapView).
- **Returns:** map instance and refs to basemap and WMS layers so parent can swap basemap and MapView can read WMS refs for GetFeatureInfo.

### Sidebar.jsx / SidebarSection

- **Responsibility:** Collapsible panel; mobile: overlay + close button. Sections: Basemap (list of options), Overlay layers (WMS checkboxes + opacity sliders + “Fit” to extent), Map tools (Info, Add, Distance + tool hint + distance result card + error). No business logic; only UI and callbacks from App.

### AppHeader.jsx

- **Responsibility:** Logo/title and, on mobile, menu button to toggle sidebar. No map or API logic.

### LocationSearch.jsx

- **Responsibility:** Controlled input, submit handler that calls `geocode(query)` and then `onSelect(result)` or sets error. No map access; parent uses result to fit view and set highlight.

### AddLocationModal.jsx

- **Responsibility:** Modal with lat/lon display, name/type inputs, submit/cancel. On submit calls `onSubmit(name, type)`; parent does `createPlace` and closes modal. Handles loading/success/error and Escape to close.

### placesApi.js

- **Responsibility:** Build full URL from config.apiBase; implement createPlace, getPlaces, getNearby, getNearest, getDistance with fetch; on non-ok parse JSON error and throw Error(message) so components can show it.
- **Config:** apiBase from `config.js` (VITE_API_BASE or empty for same-origin/proxy).

### config.js

- **Responsibility:** Export `apiBase`: from env `VITE_API_BASE`, or empty (same origin), or fallback for SSR (e.g. localhost:8080). Used by placesApi and any other API client.

### Utils

- **wmsGetFeatureInfo.js:** Builds GetFeatureInfo URL (REQUEST, SERVICE, VERSION, LAYERS, QUERY_LAYERS, INFO_FORMAT, I, J, WIDTH, HEIGHT, CRS/BBOX for 1.3.0). Used by MapView on Info-tool click.
- **osrmRoute.js:** Calls OSRM route API (driving) with two points, returns `{ coordinates, distanceMeters, durationSeconds }` or null. Used by MapView when distancePoints.length === 2.
- **geocode.js:** Calls Nominatim search, returns first result with lat, lon, displayName, bbox (converted to [w,s,e,n]). Used by LocationSearch.

---

## 6. Interview Q&A — When, How, What, Why

**Q: How do you display layers on the map?**  
We use OpenLayers with a fixed layer order: (1) one basemap TileLayer (XYZ/OSM), (2) multiple WMS TileLayers (TileWMS for caching), (3) vector layer for places (GeoJSON from our API), (4) vector layer for distance (points + route line), (5) vector layer for search highlight (dim + outline). Basemap and WMS are created in useMap; the rest in MapView. Visibility and opacity of WMS are driven by React state and synced in useMap.

**Q: Why PostGIS geography instead of geometry?**  
We need real-world distances in meters for “within 5 km” and “nearest place.” Geography type uses the ellipsoid for distance; geometry in 4326 would use degrees. We also use a GIST index on the geography column for ST_DWithin and the nearest-neighbor operator (<->).

**Q: How does the “nearest place” query work?**  
We use PostGIS’s KNN operator: `ORDER BY place.location <-> ST_SetSRID(ST_MakePoint(:lon,:lat),4326)::geography LIMIT 1`. The `<->` operator uses the spatial index (GIST) for efficient nearest-neighbor search without a fixed radius.

**Q: How is straight-line distance computed?**  
In the custom repository we run a native query: `SELECT ST_Distance( geography1, geography2 )` where each geography is built from (lon, lat) with SRID 4326. PostGIS returns meters. We expose it as distanceKm and distanceMeters in the API.

**Q: Why do you have both geographical distance and road distance?**  
Geographical distance (our API) is “as the crow flies” and is always available. Road distance (OSRM) shows driving distance and time; we draw the route on the map. So the user gets both metrics and a visual path.

**Q: How does GetFeatureInfo work?**  
When the user clicks the map with the Info tool, we take the click pixel and current map extent/size. We build a WMS GetFeatureInfo URL (LAYERS, BBOX, CRS, I, J, WIDTH, HEIGHT, INFO_FORMAT). We request GML first and parse it into a table; if that fails we try HTML and strip error blocks. We show the result in a popup overlay.

**Q: Why TileWMS instead of ImageWMS?**  
TileWMS requests small tiles that can be cached and loaded incrementally when panning/zooming; ImageWMS typically requests one image per view change, which is slower and doesn’t cache as well. So we use TileWMS for better performance and UX.

**Q: How do you handle errors from the API?**  
Backend: GlobalExceptionHandler returns a consistent ErrorResponse (status, code, message, path, optional field/fieldErrors). Validation errors (Bean Validation or CoordinateValidator) become 400; no nearest place becomes 404; the rest 500 with a generic message. Frontend: fetch in placesApi checks res.ok; on failure we parse JSON and throw Error(message) so components can show it (e.g. in sidebar or modal).

**Q: How is the frontend configured for different environments?**  
We use Vite; API base URL comes from `VITE_API_BASE`. Empty means same origin (production same-domain or dev proxy). In dev, Vite proxies `/api` to localhost:8080 so we don’t need CORS for local development. Backend CORS allows configured origins (e.g. localhost:5173, production domain).

**Q: What happens when the user adds a place?**  
User selects Add tool and clicks the map → MapView converts coordinate to lon/lat and calls onAddLocationClick(lat, lon) → App opens AddLocationModal with that lat/lon. User enters name/type and submits → App calls createPlace({ name, type, lat, lon }) → POST /api/places → backend validates, saves Place with PostGIS point, returns GeoJSON Feature → App closes modal and increments placesRefreshTrigger → MapView’s effect runs and calls getPlaces() → replaces features in the places VectorSource so the new marker appears.

**Q: Why Liquibase instead of Hibernate ddl-auto?**  
We want versioned, repeatable schema changes (PostGIS, table, indexes) and no automatic schema changes in production. Liquibase changeSets are applied in order and tracked; ddl-auto is not suitable for production and doesn’t express indexes and extensions clearly.

**Q: How do you secure the API?**  
For this project we didn’t add authentication; the API is public. We use Spring Security to (1) allow only the listed place endpoints under /api, (2) disable CSRF for stateless API, (3) configure CORS from application config so the frontend origin is allowed. For a real product we’d add JWT or session-based auth and scope endpoints by role.

---

You can use this guide to walk through the codebase and answer “how did you build X?” or “why did you choose Y?” in interviews. For exact request/response shapes, see **API.md**.
