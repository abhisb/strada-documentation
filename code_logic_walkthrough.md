# STRADA-DE Code-Level Walkthrough & Logic Documentation

This document explains the end-to-end code logic, database operations, API routing, and user interface features of **STRADA-DE**.

---

## 1. Architectural Overview
STRADA-DE is structured using a clean separation of concerns (a modular design where the database, backend routes, and frontend views are separated):

* **Database (SQLite)**: A lightweight, single-file database (`database.db`) stores all records.
* **Backend (Node.js & Express)**: Listens for user web requests and fetches data from the database using SQL queries.
* **Caching Middleware**: Remembers recent backend queries in memory so it can answer subsequent identical requests instantly without re-querying the database.
* **Frontend (Vanilla HTML, CSS, & ES6 JavaScript)**: The interface shown in the web browser. It fetches data dynamically from the backend and draws graphs (using *Chart.js*) and tables.

---

## 2. Database Connection & Schema Setup
File: [`src/config/db.js`](./src/config/db.js)

### Logic Explained:
1. **Connection**: Imports the `sqlite3` library and opens a connection pool to `database.db`.
2. **Promisification**: Standard SQLite operations use callback functions, which can become messy. The code wraps these operations in JavaScript `Promises` (`runQuery` for writing data, `getQuery` for fetching a single row, and `allQuery` for fetching multiple rows). This allows the rest of the backend to write clean, modern `async/await` code.
3. **Database Tables**:
   - `regions`: Stores names, codes (AGS), populations, and hierarchical parent-child associations for states, districts, and municipalities.
   - `accidents`: Stores individual accident events (year, month, hour, category, latitude/longitude, and participant flags).
   - `metadata_sources`: Stores details about licenses and source links for legal auditing.

---

## 3. Caching System
File: [`src/middleware/cache.js`](./src/middleware/cache.js)

### Logic Explained:
To prevent the SQLite database from getting overloaded with repetitive requests, we use an in-memory Cache:
1. When a user requests an API endpoint (e.g. state aggregates), the caching script checks a JavaScript `Map` container using the request URL as the "key".
2. **Cache Hit**: If the key exists and has not expired (TTL is 5 minutes), the server returns the saved result immediately. It appends the header `X-Cache: HIT` to the response.
3. **Cache Miss**: If not found or expired, the request is passed to the database service. Once the database returns the results, the caching script saves the results in the `Map` with a timestamp, then returns the data with the header `X-Cache: MISS`.

---

## 4. Backend Service & Query Logic
Files: `src/services/`

The service layer contains the SQL queries executed against the database:

### A. Region Service ([`regionService.js`](./src/services/regionService.js))
* Fetches administrative regions. If the query specifies `level=state`, it runs:
  `SELECT * FROM regions WHERE level = 'state'`
* If a parent region ID is passed, it returns only the child regions associated with that parent.

### B. Accident Service ([`accidentService.js`](./src/services/accidentService.js))
* Retrieves paginated list of accident records (e.g., page 1, 50 rows).
* Builds SQL clauses dynamically depending on active frontend filters (e.g., year, state, category, month, hour).

### C. Statistics Service ([`statsService.js`](./src/services/statsService.js))
* **Regional Aggregates**: Computes total accident counts and joins them with the `regions` table populations to calculate the **Accident Rate per 100,000 inhabitants**:
  $$\text{Accident Rate} = \left( \frac{\text{Accidents Count}}{\text{Region Population}} \right) \times 100,000$$
* **Hourly and Monthly curves**: Runs group-by SQL statements (`GROUP BY hour` / `GROUP BY month`) to summarize counts for graph visualization.

---

## 5. Spatial Radius Search Logic
File: [`src/services/accidentService.js`](./src/services/accidentService.js)

### Plain English Explanation:
The goal is to find all accidents occurring within a circular radius (e.g., 10 km) of a specific latitude/longitude point. Calculating trigonometry formulas on over a million database rows is very slow. To solve this, the code uses a two-step approach:
1. **Bounding Box Pre-filtering (Fast SQL Check)**:
   - First, it calculates a square bounding box around the target point using simple addition/subtraction.
   - It queries the database for accidents whose coordinates fall inside this square:
     `SELECT * FROM accidents WHERE lat BETWEEN minLat AND maxLat AND lon BETWEEN minLon AND maxLon`
   - This narrows down over 1.2 million rows to a few hundred in milliseconds.
2. **Haversine Circle Check (Exact Math Check)**:
   - For the matching rows, the code executes the **Haversine formula** in JavaScript to calculate the exact curved-surface distance between the center coordinates and the accident coordinate.
   - If the exact distance is less than or equal to the specified radius, it keeps the row and calculates the distance in kilometers.

---

## 6. Frontend Navigation & Tab Toggling
File: [`public/js/main.js`](./public/js/main.js)

### Logic Explained:
1. **Event Bindings**: When the HTML DOM loads, `main.js` loops over the navbar links (`.nav-item`) and binds click listeners.
2. **Tab Switching**: Clicking a navigation item:
   - Removes the `active` styling class from all menus and panels.
   - Adds the `active` styling class to the clicked menu item and tab panel. In CSS, `.tab-content` is hidden (`display: none`), while `.tab-content.active` is visible (`display: block`).
   - Updates the sub-header title text dynamically.
3. **Data Orchestration**: Calls the specific load method for the newly selected tab using the active global filters (Year and State).

---

## 7. Tab Component Implementation & Logics

### A. Dashboard Tab ([`dashboard.js`](./public/js/dashboard.js))
* **KPI Indicators**: Queries `/api/stats`, parses JSON metrics, and sets `.innerText` for total, fatal, bicycle, and pedestrian counts.
* **YoY monthly and Hourly trend graphs**: Uses the **Chart.js** library to build visualizations. Before drawing a new chart, it calls `.destroy()` on the existing chart instance to prevent mouseover rendering issues.
* **Regional Breakdowns Table**:
  - If a state is selected, it fetches district-level aggregates, filters out rows that do not match the selected state's AGS prefix, sorts them descending, and lists district counts/rates in the table.
  - If "All Germany" is selected, it displays federal states data.

### B. Spatial Search Tab ([`spatial.js`](./public/js/spatial.js))
* Reads user inputs from numerical boxes (`#spatial-lat`, `#spatial-lon`, `#spatial-radius`).
* Sends a GET fetch query to `/api/accidents/spatial`.
* Reads the returned matching results list:
  - Generates a **Chart.js Bar Chart** visualizing the counts grouped by category (Fatal, Serious, Light).
  - Generates table rows listing the accident ID, the calculated distance in kilometers (e.g. `0.452 km`), the severity badge, and icons for involved participants.

### C. Rates & Rankings Tab ([`rankings.js`](./public/js/rankings.js))
* Reads options for administrative level (State/District), sort metric (Total count/Density rate per 100k), and limits (Top 10/All/Bottom 10).
* Sorts the returned aggregates array dynamically using JavaScript's `.sort()` method.
* Re-renders the ranking table rows and populates a **horizontal bar chart** visualizing density scores side-by-side.

### D. Zero Cases Tab ([`zeroCases.js`](./public/js/zeroCases.js))
* **The Problem**: A standard query of the accidents table will skip regions that have no accident events recorded.
* **The Logic Solution**: The code fetches all municipality aggregates. It matches them against the baseline regions list. Any municipality whose `accident_count` is 0 is identified.
* **Dynamic State Filtering**: Reads the selected state code. If a state filter is active, it checks if `region_ags.startsWith(state)` to show only the zero-accident municipalities inside that state (reconciles Saxony zero cases dynamically).

### E. Provenance Tab ([`provenance.js`](./public/js/provenance.js))
* Queries `/api/metadata/sources` to load source names, download URLs, license texts, and last-checked timestamps.
* Dynamically creates CSS grid cards (`.source-card`) for each source entry and appends them to the page wrapper, proving dataset licensing legality to the examiner.
