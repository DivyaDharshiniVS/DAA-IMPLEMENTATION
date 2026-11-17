# Geographic Nearest Neighbor Finder

> *"Sometimes what we're looking for is closer than we think"*

A research-grade Python implementation of **2D $K$-Dimensional Tree ($K\text{D}$-Tree) spatial indexing**, **Haversine geodetic spherical trigonometry**, and **branch-and-bound nearest neighbor search** tailored for geographic information systems (GIS).

The entire application runs from a single, self-contained Python script (`main.py`) using only standard libraries plus `folium`, computing all DAA algorithms in Python and exporting a self-contained, interactive `index.html` research dashboard ready for 1-click static deployment on **GitHub Pages**.

---

## Table of Contents
1. [Theoretical Algorithmic Foundations](#theoretical-algorithmic-foundations)
2. [Mathematical Formulations](#mathematical-formulations)
   - [Divide & Conquer Recurrence & Master Theorem](#1-divide--conquer-recurrence--master-theorem)
   - [Orthogonal Hyperplane Spatial Bisection](#2-orthogonal-hyperplane-spatial-bisection)
   - [Branch-and-Bound Pruning Mechanics](#3-branch-and-bound-pruning-mechanics)
   - [Haversine Spherical Geodesic](#4-haversine-spherical-geodesic)
3. [Architecture & DAA Modules](#architecture--daa-modules)
   - [Module 1: Location Data Loader (GPS)](#module-1-location-data-loader-gps)
   - [Module 2: KD-Tree Construction (Divide & Conquer)](#module-2-kd-tree-construction-divide--conquer)
   - [Module 3: Nearest Neighbor & Radius Benchmark](#module-3-nearest-neighbor--radius-benchmark)
   - [Module 4: Interactive Folium Map & Real-Time Query](#module-4-interactive-folium-map--real-time-query)
   - [Module 5: Source Code Workspace](#module-5-source-code-workspace)
4. [Empirical Benchmarking Results](#empirical-benchmarking-results)
5. [Installation & Execution](#installation--execution)
6. [1-Click GitHub Pages Deployment](#1-click-github-pages-deployment)
7. [Repository Structure](#repository-structure)

---

## Theoretical Algorithmic Foundations

In Geographic Information Systems (GIS) and location-based services (LBS), finding the $K$ closest facilities (e.g., hospitals, pharmacies, transit stations) to an arbitrary GPS query coordinate is a foundational spatial problem.

A naive **Brute-Force Linear Scan** computes the distance between the query point and every point in the dataset:
- **Time Complexity:** $\mathcal{O}(N)$ distance calculations per query.
- For high-throughput spatial querying (e.g., autonomous ride routing, emergency dispatch), $\mathcal{O}(N)$ distance evaluations on millions of points create severe latency bottlenecks.

The **2D KD-Tree** solves this challenge through recursive **Divide & Conquer**:
- Spatial points are hierarchically organized into a balanced binary search tree where every level bisects space along alternating orthogonal dimensions (Latitude and Longitude).
- During search, geometric bounding tests allow the algorithm to **prune entire subtrees** whose bounding hyperplanes lie farther away than the current $K$-th best candidate.
- **Average Query Complexity:** $\mathcal{O}(\log N)$, reducing distance evaluations by **over 90%** on realistic geographic datasets.

---

## Mathematical Formulations

### 1. Divide & Conquer Recurrence & Master Theorem

To construct a perfectly balanced 2D KD-Tree, the algorithm recursively computes the median along the splitting axis:
1. At depth $d$, sort the $N$ points along axis $k = d \pmod 2$.
2. The median element becomes the root node of the current subtree.
3. Left slice points ($< \text{median}$) form the left subtree; right slice points ($> \text{median}$) form the right subtree.

The computational recurrence relation is:
$$T(N) = 2 \cdot T\left(\frac{N}{2}\right) + \mathcal{O}(N)$$

By the **Master Theorem** ($a = 2$, $b = 2$, $f(N) = \mathcal{O}(N)$):
$$\log_b a = \log_2 2 = 1 \implies f(N) = \Theta(N^{\log_b a}) = \Theta(N)$$
$$\therefore T(N) = \Theta(N \log N)$$

The resulting tree height is strictly bounded:
$$h = \lfloor \log_2 N \rfloor$$
For $N = 250$, the theoretical optimal height is $\lfloor \log_2 250 \rfloor = 7$ (maximum depth 8 levels).

---

### 2. Orthogonal Hyperplane Spatial Bisection

A 2D KD-Tree alternates splitting axes cyclically at each depth level:
$$\text{Axis}(d) = \begin{cases} 0 \quad (\text{Latitude, horizontal hyperplane}), & \text{if } d \equiv 0 \pmod 2 \\ 1 \quad (\text{Longitude, vertical hyperplane}), & \text{if } d \equiv 1 \pmod 2 \end{cases}$$

Each node $u$ defines an orthogonal bounding box interval:
$$\text{BBox}(u) = [\phi_{\min}, \lambda_{\min}, \phi_{\max}, \lambda_{\max}]$$
This guarantees that subtrees correspond to disjoint rectangular bounding boxes in geographic coordinate space.

---

### 3. Branch-and-Bound Pruning Mechanics

During a $K$-Nearest Neighbor (KNN) query, candidate points are maintained in a **Bounded Max-Heap** of capacity $K$ ordered by distance:
- The top of the heap holds the current $K$-th farthest neighbor ($r_{\text{worst}}$).
- The traversal visits the "near" subtree first (where the query coordinate lies).
- Before visiting the "far" subtree, the algorithm computes the minimum great-circle perpendicular distance $d_\perp$ from the query point $q = (\phi_q, \lambda_q)$ to the splitting hyperplane $H$:

$$d_\perp(q, H) = \begin{cases} 
R \cdot |\phi_q - \phi_{\text{split}}| \cdot \frac{\pi}{180}, & \text{if } \text{Axis} = 0 \text{ (Latitude)} \\ 
R \cdot \cos\left(\phi_q \cdot \frac{\pi}{180}\right) \cdot |\Delta\lambda| \cdot \frac{\pi}{180}, & \text{if } \text{Axis} = 1 \text{ (Longitude)} 
\end{cases}$$

where $|\Delta\lambda| = \min(|\lambda_q - \lambda_{\text{split}}|, 360^\circ - |\lambda_q - \lambda_{\text{split}}|)$, and $R = 6371.0088\text{ km}$.

**Pruning Decision Rule:**
$$\text{If } |\text{Heap}| = K \quad \text{AND} \quad d_\perp(q, H) \ge r_{\text{worst}} \implies \mathbf{PRUNE}(\text{FarSubtree})$$

The entire opposite branch is discarded because **no point within that subtree can physically lie closer than the current $K$ best candidates**.

---

### 4. Haversine Spherical Geodesic

Flat Euclidean distance ($\sqrt{\Delta x^2 + \Delta y^2}$) causes severe geometric distortion away from the equator because meridians converge as $\cos(\phi)$. 

At latitude $\phi = 13.08^\circ\text{ N}$ (Chennai), $1^\circ$ of longitude equals $\approx 108.4\text{ km}$, while $1^\circ$ of latitude equals $\approx 110.6\text{ km}$. Flat planar projections introduce up to **15% positional ranking errors**.

The **Haversine formula** provides geodetically exact great-circle distance on the WGS-84 mean volumetric Earth sphere:
$$\Delta\phi = \phi_2 - \phi_1, \quad \Delta\lambda = \lambda_2 - \lambda_1$$
$$a = \sin^2\left(\frac{\Delta\phi}{2}\right) + \cos(\phi_1)\cos(\phi_2)\sin^2\left(\frac{\Delta\lambda}{2}\right)$$
$$c = 2 \cdot \text{atan2}\left(\sqrt{a}, \sqrt{1 - a}\right)$$
$$d = R \cdot c \quad (R = 6371.0088\text{ km})$$

---

## Architecture & DAA Modules

```
+-----------------------------------------------------------------------------------------+
|                                    main.py (Python)                                     |
|  - Data Generation (250 Chennai POIs across 4 categories)                               |
|  - Recursive 2D KD-Tree Construction & Spatial Bounding Boxes                           |
|  - Haversine Geodesic & Orthogonal Perpendicular Distance Functions                    |
|  - Bounded Max-Heap KNN Search + Pruning vs. Baseline Linear Scan Benchmark             |
|  - Interactive SVG Binary Tree Renderer & CartoDB Dark Matter Folium Map Engine         |
+-----------------------------------------------------------------------------------------+
                                           |
                                  Generates / Exports
                                           v
+-----------------------------------------------------------------------------------------+
|                                  index.html (Dashboard)                                 |
|  - Research-Grade UI: Deep Emerald (#092D25) & Champagne Gold (#D9B66F)                 |
|  - Tab 1: SOURCE CODE (Syntax-highlighted Python + Copy button + DAA Complexity)        |
|  - Tab 2: Location Data Loader (GPS POI Registry, Presets, Live Search Filter)          |
|  - Tab 3: KD-Tree Construction (Spatial Index Metrics, Interactive SVG, 2D Canvas)      |
|  - Tab 4: Nearest Neighbor & Benchmark (Side-by-side performance cards, Traversal log) |
|  - Tab 5: Interactive Map & Query (Folium dark map, Sliders, Category filter, Vectors)  |
|  - Client-Side JS Mirror: 60 FPS real-time recalculation, map-click repositioning       |
+-----------------------------------------------------------------------------------------+
```

### Module 1: Location Data Loader (GPS)
- Curates $N = 250$ realistic GPS Points of Interest across metropolitan Chennai ($12.95^\circ - 13.13^\circ\text{ N}, 80.18^\circ - 80.30^\circ\text{ E}$).
- 4 facility categories: `Hospital` (Red), `Restaurant` (Amber), `Pharmacy` (Emerald), and `Transit Hub` (Blue).
- Quick reference preset chips for instant relocation: **Chennai Central**, **Anna Nagar**, **T. Nagar**, **Adyar**, **Marina Beach**, and **Guindy Hub**.
- Tabular registry with live text filtering and "Query Near This" button.

### Module 2: KD-Tree Construction (Divide & Conquer)
- Pure Python 2D KD-Tree built recursively using median bisection.
- Spatial index metrics readout: Total Nodes, Tree Depth, Construction Time (in milliseconds), and Splitting Dimensions.
- Responsive **SVG binary tree visualization** displaying the top levels of the index with node coordinates, splitting axes, median POI names, and interactive hover tooltips.
- **2D Hyperplane Partitioning Diagram** showing recursive horizontal and vertical bounding lines across the geographic map extent.

### Module 3: Nearest Neighbor & Radius Benchmark
- Real-time performance comparison between KD-Tree search and Brute-Force Linear Scan.
- Tracks:
  * Exact distance calculations performed
  * Query execution latency ($\text{ms}$)
  * Pruned search space percentage ($\%$)
  * Computational speedup factor ($X\times$)
- **Active Query Branch Pruning Trace**: Step-by-step inspector showing which branches were explored and which were pruned.

### Module 4: Interactive Folium Map & Real-Time Query
- Dark-mode OpenStreetMap using **CartoDB Dark Matter** tiles matching the Deep Emerald & Gold theme.
- Champagne Gold pulsing query anchor pin.
- Dashed Champagne Gold search horizon boundary circle.
- Highlighted nearest neighbors with rank badges (`#1`, `#2`, ...), connecting dashed distance vectors, and rich popups.
- Controls: Sliders for Latitude, Longitude, $K$ (1 to 25), Radius ($0.5$ to $20\text{ km}$), and Category filter buttons with active gold aura.
- **Map Click Mode:** Click anywhere on the map to instantly relocate the query point.

### Module 5: Source Code Workspace
- Displays the complete `main.py` source code in a syntax-highlighted viewer.
- 1-click **Copy Code** button with active state transition ("✓ Copied!").
- Concise theoretical explanation of Master Theorem recurrence, balanced tree invariants, and geodetic bounding accuracy.

---

## Empirical Benchmarking Results

Execution results on an anchor query at **Chennai Central Railway Station** ($13.0827^\circ\text{ N}, 80.2707^\circ\text{ E}$, $K = 5$ nearest neighbors):

| Performance Dimension | KD-Tree (Divide & Conquer) | Brute-Force Linear Scan | Efficiency Advantage |
| :--- | :---: | :---: | :---: |
| **Distance Calculations** | **22 computations** | **250 computations** | **91.20% fewer operations** |
| **Query Latency** | **0.183 ms** | **0.600 ms** | **3.28× faster execution** |
| **Distance Speedup** | **11.36× fewer calcs** | Baseline ($1.0\times$) | **Significant algorithmic reduction** |
| **Average Complexity** | $\mathcal{O}(\log N)$ | $\mathcal{O}(N)$ | Logarithmic scaling guarantee |
| **Worst-Case Complexity** | $\mathcal{O}(\sqrt{N})$ in 2D | $\mathcal{O}(N)$ | Sub-linear bound |
| **Nearest Neighbor Match** | Top 5 Identified | Top 5 Identified | **100.0% Exact Mathematical Equivalence** |

### Top 5 Discovered Nearest Neighbors to Chennai Central:
1. **Puratchi Thalaivar MGR Central Railway Station** (`Transit Hub`) — $0.000\text{ km}$ ($0\text{ m}$)
2. **Sangeetha Fast Foods - George #30** (`Restaurant`) — $0.170\text{ km}$ ($170\text{ m}$), Bearing: $6.6^\circ\text{ N}$
3. **Hotel Saravana Bhavan** (`Restaurant`) — $0.472\text{ km}$ ($472\text{ m}$), Bearing: $99.5^\circ\text{ E}$
4. **Apollo Pharmacy Central Concourse** (`Pharmacy`) — $0.574\text{ km}$ ($574\text{ m}$), Bearing: $92.2^\circ\text{ E}$
5. **Buhari Hotel (Since 1951)** (`Restaurant`) — $0.608\text{ km}$ ($608\text{ m}$), Bearing: $70.8^\circ\text{ E}$

---

## Installation & Execution

### Prerequisites
- Python 3.10, 3.11, 3.12, or 3.13
- Internet access (for downloading the `folium` package once)

### 1. Install Dependencies
```bash
pip install -r requirements.txt
```
*(Only `folium` is required; all other algorithms use Python's built-in `math`, `heapq`, `random`, `time`, and `json` libraries).*

### 2. Execute the Pipeline
```bash
python main.py
```

### 3. Expected Console Output
```text
================================================================================
 GEOGRAPHIC NEAREST NEIGHBOR FINDER (DAA SPATIAL COMPUTING PROJECT)
 'Divide and conquer spatial data to uncover the closest destinations in logarithmic time.'
================================================================================

[*] Module 1: Generating realistic GPS Points of Interest across Chennai (N = 250)...
    [+] Successfully generated 250 POIs across 4 categories.
        - Hospital    :  60 POIs ( 24.0%)
        - Pharmacy    :  56 POIs ( 22.4%)
        - Restaurant  :  86 POIs ( 34.4%)
        - Transit Hub :  48 POIs ( 19.2%)

[*] Module 2: Constructing 2D KD-Tree via recursive Divide & Conquer...
    [+] KD-Tree Constructed in: 1.162 ms
    [+] Total Indexed Nodes:    250
    [+] Maximum Tree Depth:     8 levels (Theoretical optimal: 7)
    [+] Alternating Dimensions: Axis 0 = Latitude, Axis 1 = Longitude

[*] Module 3: Executing DAA Spatial Benchmarks for anchor: 'Chennai Central'...
    Coordinates: Lat 13.0827, Lon 80.2707
    [+] KNN Query (K = 5):
        * KD-Tree Calculations:     22 distance computations
        * Linear Scan Calculations: 250 distance computations
        * Pruned Search Space:      91.20% of subtrees eliminated
        * Distance Speedup Factor:  11.36x fewer computations
        * KD-Tree Latency:          0.1832 ms
        * Linear Scan Latency:      0.6002 ms
    [+] RESEARCH-GRADE VERIFICATION: KD-Tree exactly matches 100% of linear scan results!

[*] Module 4 & 5: Exporting self-contained index.html (Folium Dark Map + Bespoke Research Dashboard)...
    [+] SUCCESS: Exported '.../index.html' (249.6 KB)
    [+] Deployment Ready: Ready for 1-click static deployment on GitHub Pages.
================================================================================
```

### 4. View the Interactive Dashboard
Double-click `index.html` to open it in any web browser (Chrome, Edge, Firefox, Safari) or serve it locally:
```bash
python -m http.server 8000
```
Then navigate to `http://localhost:8000`.

---

## 1-Click GitHub Pages Deployment

The generated `index.html` is completely self-contained (all stylesheets, vector maps, Leaflet dependencies, and client-side KD-Tree algorithms are bundled within the single file).

To deploy live on the web:
1. Initialize a Git repository and commit the files:
   ```bash
   git init
   git add .
   git commit -m "feat: Geographic Nearest Neighbor Finder DAA Project"
   ```
2. Push to your GitHub repository:
   ```bash
   git branch -M main
   git remote add origin https://github.com/<your-username>/<repo-name>.git
   git push -u origin main
   ```
3. In your GitHub repository:
   - Go to **Settings** $\rightarrow$ **Pages**.
   - Under **Build and deployment** $\rightarrow$ **Source**, select **Deploy from a branch**.
   - Under **Branch**, select `main` and folder `/(root)`, then click **Save**.
4. Within 30 seconds, your project is live at:
   `https://<your-username>.github.io/<repo-name>/`

---

## Repository Structure

```
DAA OUTPUT/
├── main.py              # Master script: DAA algorithms, benchmarks, Folium map & HTML compiler
├── requirements.txt     # Python dependency definition (folium>=0.15.0)
├── index.html           # Self-contained export: Folium dark map + research dashboard
└── README.md            # Comprehensive research and algorithmic documentation
```

---

## Academic Citation & Integrity
Developed for academic study and evaluation in **Design and Analysis of Algorithms (DAA)**. Implements classical algorithmic paradigms (Divide & Conquer, Recursive Partitioning, Bounded Priority Queues, Spatial Hyperplane Pruning) applied to real-world spherical geodetic computing.
