# Implementation Plan - STRADA-DE Road Accident Analytics

This document details the modular system implementation for the **STRADA-DE (Straßenverkehrsunfall-Analyse-Portal Deutschland)** platform. It integrates official German road accident data (Unfallatlas) and regional statistics (states, districts, and populations) into a reusable own data service, complete with a Node.js/SQLite backend, a documented REST API, and a clean, responsive HTML/JS web interface.

The design replaces typical dark-neon templates with a professional, light-themed academic layout matching official German federal statistical portals (e.g. Destatis), ensuring a hand-crafted look.

---

## 1. External Data Integration Details
The project integrates the following external data sources:
1. **Unfallatlas (Verkehrsunfälle mit Personenschaden)**: Point-data CSV files (2020–2024) downloaded and parsed automatically.
2. **OpenPLZ API**: REST API query interface used to dynamically fetch administrative region definitions (states, districts, and Saxony municipalities).
3. **RKI Census Data & Kraftfahrt-Bundesamt (KBA)**: Local files and GitHub streams providing populations and passenger car vehicle stocks (used as indicators for rate calculations).

---

## 2. Project Folder & MVC Architecture
The codebase is structured modularly:
```text
semester project/
  ├── package.json
  ├── server.js (Minimal Express router bootstrapper)
  ├── src/
  │    ├── config/
  │    │    └── db.js (SQLite connection pool & migrations)
  │    ├── middleware/
  │    │    └── cache.js (In-memory GET endpoint caching TTL middleware)
  │    ├── services/
  │    │    ├── regionService.js (Regions SQL queries)
  │    │    ├── accidentService.js (Accidents & Spatial queries)
  │    │    └── statsService.js (Aggregations, QA Solvers, and Logs)
  │    └── routes/
  │         ├── regions.js
  │         ├── accidents.js
  │         ├── stats.js
  │         └── questions.js
  ├── public/
  │    ├── index.html (Destatis-style light White dashboard template)
  │    ├── style.css (Vanilla layout tokens, menus, and typography)
  │    └── js/
  │         ├── main.js (State controller & tab navigation)
  │         ├── dashboard.js (KPI cards, YoY bars, doughnuts, and regional breakdowns)
  │         ├── spatial.js (Haversine coordinate radius search and severity charts)
  │         ├── rankings.js (Level sorting tables and bar charts)
  │         ├── zeroCases.js (Sachsen zero cases reports)
  │         └── provenance.js (Metadata grids loader)
  └── database.db
```

---

## 3. Express Server API Enhancements

#### `server.js` (Root Entry Point)
Exposes the required REST API endpoints and mounts modular routers under `/api/*`:
* **API Error Handling**: Centralized error middleware captures exceptions and returns standardized JSON error response payloads with matching HTTP status codes (400, 404, 500).
* **Caching Strategy**: Custom cache middleware intercepts GET requests and returns in-memory cached responses to optimize performance, appending an `X-Cache: HIT/MISS` response header.
* **Spatial Query Endpoint**: `GET /api/accidents/spatial` uses bounding-box filters and the Haversine formula on coordinates to filter matching accident occurrences within a radial kilometer range of a lat/lon center.
* **OpenAPI Spec**: Serves a full OpenAPI 3.0 specification at `GET /api/openapi.json`.

---

## 4. Frontend Web Dashboard & Features

#### `public/index.html` & `public/style.css`
A premium light-themed UI dashboard utilizing Google Fonts (Outfit) and FontAwesome icons, structured around five tabs:
1. **Dashboard Overview**: Key KPIs (Total, Fatal, Bike, Pedestrian), YoY monthly trends (2023 vs 2024 comparison), Hourly curves, and road user breakdown. Features a **Regional Accident Breakdowns** table displaying state/district aggregates depending on filters.
2. **Spatial Search**: Radial query interface allowing users to input Latitude, Longitude, and Radius (km) to list matching accident events alongside a Bar chart mapping severities inside the range.
3. **Rates & Rankings**: Visualizing accident rates per 100,000 inhabitants. Supports ranking of top/bottom districts and comparison across regions.
4. **Zero Cases Report**: Dynamically queries and lists municipalities in the selected state (utilizing full region reference baseline) that recorded zero accident events.
5. **Data Provenance**: Displays metadata provenance, licenses, and legal urls for integrated datasets.

---

## 5. Verification Plan

### Automated Tests
* A script `validate.js` in the workspace root executes queries corresponding to the 5+2 mandatory examiner questions and logs the results.

### Manual Verification
* Boot the backend server using `node server.js` and open `http://localhost:3005/` in the browser.
* Use the **Dashboard** and **Rates & Rankings** controls to verify that counts, tables, and Chart.js graphics update correctly when state/year filters change.
* Test the **Spatial Search** query by inputting coordinates (`lat: 51.104`, `lon: 13.201`, `radius: 10km`) and verifying that matching records are displayed with correct Haversine distance.
* Test the **Zero Cases Report** to check dynamic municipality counts for Saxony.
