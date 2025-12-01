# README — Technical Guide to the Packaging Optimisation Solution

This document provides a detailed explanation of the `Optimised_boxes_solution.py` script. The script defines a percentile-based optimisation code that determines an optimal set of five box dimensions. The objective is to minimise void fill while limiting the number of outlier SKUs that would require an oversized "safety box."


### How to run the optimiser code 

**Input file**: a returns item dataset in CSV format, containing one row per SKU with the columns sku_id, l, w, h, and quantity.
- Dimensions (l, w, h) must be in millimetres.
- quantity must be an integer representing the number of returned units for that SKU.
  
**Launch**: run the optimiser script directly from a terminal and provide the path to your returns dataset when requested.
**Output**: the console displays the proposed set of five optimised boxes, any safety box added for outliers, and a summary report of usage and average void fill, including metrics with and without the safety box.

---

## Part I: Methodological framework

### 1.1. Problem modelling using Class 
The core of the script is structured around three main classes that model the problem:

- **`SKU`**: Represents a single stock-keeping unit. Each instance holds its dimensions (sorted from largest to smallest for orientation-independent fit assessment), volume, and quantity. It also tracks which box it has been assigned to.

- **`Box`**: Represents a type of packaging box. Similar to `SKU`, its dimensions are sorted. It includes essential methods to check if a SKU can fit inside (`can_fit`) and to calculate the resulting void fill percentage (`calculate_void_fill`).

- **`BoxOptimiser`**: The central class that orchestrates the optimisation process. It takes a list of SKU objects as input and manages the generation, evaluation, and selection of candidate box configurations.

### 1.2. Optimisation approach
The algorithm implements a **grid search** over a defined set of percentiles. 

The process is as follows:
1. **Candidate generation**: For each percentile value in a predefined grid (e.g., 86, 88, ..., 94), a unique five-box configuration is generated.
2. **Evaluation**: Each configuration is evaluated by simulating the assignment of all SKUs. This evaluation yields two key metrics: the average void fill (weighted by quantity) and the rate of outliers (items that fit in none of the five boxes).
3. **Selection**: The "best" configuration is selected using an objective function that combines these two metrics.

---

## Part II: Detailed solving methodology

### 2.1. New optimsed boxes collection generation (`_create_boxes`)
This method is key to the creation of candidate solutions. The process is as follows:

- **Data weighting**: First, the SKU dataset is weighted by quantity. This means the dimensions of high-volume items are repeated proportionally, ensuring their influence on the statistical distribution is accurate.

- **Volume partitioning**: The weighted SKUs are then divided into five groups (quantiles) based on their volume. Each group thereby represents a slice of the item volume distribution.

- **Percentile-based calculation**: For each group, the dimensions of the corresponding box are determined by calculating a specified percentile (e.g., the 90th percentile) of the dimensions (`dim_1`, `dim_2`, `dim_3`) of the SKUs within that group. Using a percentile (rather than the maximum value) is designed to create boxes that fit the majority of items in a group, while accepting that a few exceptionally large items may become outliers.

- **Rounding**: The calculated dimensions are rounded up to the nearest 10mm for practical feasibility and standardisation.

```python
def _create_boxes(self, n_boxes: int, percentile: int) -> List[Box]:
    weighted = [[s.dim_1, s.dim_2, s.dim_3, s.volume] for s in self.skus for _ in range(s.quantity)]
    df = pd.DataFrame(weighted, columns=["dim_1", "dim_2", "dim_3", "volume"])
    df["group"] = pd.qcut(df["volume"], q=n_boxes, labels=False, duplicates="drop")

    boxes = []
    for gid in sorted(df["group"].unique()):
        g = df[df["group"] == gid]
        dim1 = g["dim_1"].quantile(percentile / 100.0)
        dim2 = g["dim_2"].quantile(percentile / 100.0)
        dim3 = g["dim_3"].quantile(percentile / 100.0)
        dim1 = max(int(math.ceil(dim1 / 10.0) * 10), 200)
        dim2 = max(int(math.ceil(dim2 / 10.0) * 10), 100)
        dim3 = max(int(math.ceil(dim3 / 10.0) * 10), 50)
        boxes.append(Box(f"OPT_{len(boxes)+1}", dim1, dim2, dim3))
    return boxes
```

---

### 2.2. Evaluation and objective function (`_evaluate_boxes`, `_objective`)
Once a five-box configuration is created, it must be evaluated:

- **Assignment**: Each SKU is assigned to the smallest available box (sorted by volume) into which it can fit. If it fits in none, it is marked as an outlier.

- **Metric calculation**: The script calculates the weighted average void fill for all assigned items and the percentage of outliers relative to the total item quantity.

- **Objective function**: The performance of the configuration is calculated with the following score:

  Score = Void Fill (%) + λ x Outlier Rate (%) 

  The λ (lambda) coefficient is a parameter that serves as a penalty factor. A high λ value heavily penalises configurations that produce many outliers, thus steering the optimisation toward more inclusive solutions at the cost of a potentially higher void fill.

```python
def _evaluate_boxes(self, boxes: List[Box]) -> Tuple[float, float]:
    ordered = sorted(boxes, key=lambda b: b.volume)
    for s in self.skus:
        s.assigned_box = None
        dims = s.get_dimensions()
        for b in ordered:
            if b.can_fit(dims):
                s.assigned_box = b.box_id
                break

    total = sum(s.quantity for s in self.skus)
    outliers = sum(s.quantity for s in self.skus if s.assigned_box is None)

    vf_sum = 0.0
    fit_q = 0
    for s in self.skus:
        if s.assigned_box:
            b = next(bb for bb in boxes if bb.box_id == s.assigned_box)
            vf_sum += b.calculate_void_fill(s.volume) * s.quantity
            fit_q += s.quantity

    avg_vf = (vf_sum / fit_q) if fit_q else 100.0
    out_rate = (outliers / total) * 100 if total else 0.0
    return avg_vf, out_rate
```


---


### 2.3. Outlier (`_add_safety_box_to`)
If the selected configuration produces outliers, an optional "safety box" can be added. Its dimensions are calculated to be just large enough to accommodate the largest of the outlier items. The evaluation is then re-run with this additional box to obtain final metrics that reflect a 100% fulfilment solution.

```python
def _add_safety_box_to(self, boxes: List[Box]) -> None:
    outliers = [s for s in self.skus if s.assigned_box is None]
    if not outliers:
        return
    d1 = int(math.ceil(max(s.dim_1 for s in outliers)/50)*50)
    d2 = int(math.ceil(max(s.dim_2 for s in outliers)/50)*50)
    d3 = int(math.ceil(max(s.dim_3 for s in outliers)/50)*50)
    boxes.append(Box("OPT_SAFETY", d1, d2, d3))
```

---

## Part III: Outputs and reporting

The main `optimise_packaging()` function executes the full process and generates the following outputs:

- **Console summary**: Prints a summary of key results, including average void fill and outlier rate both *with* and *without* the safety box.
- **Box dimensions**: Displays the dimensions of each optimised box.
- **Per-box statistics**: A pandas DataFrame containing, for each box, its usage rate and average void fill. This DataFrame is structured for direct use in the Streamlit visualisation dashboard.

```python
def optimise_packaging(datafile: str) -> Dict:
    df = pd.read_csv(datafile, dtype={"sku_id": "string"})
    skus = [SKU(str(r.sku_id), int(r.l), int(r.w), int(r.h), int(r.quantity)) for _, r in df.iterrows()]

    opt = BoxOptimiser(skus)
    best = opt._optimise()

    vf_with_safety, out_with_safety = opt._evaluate_boxes(best["boxes"])

    total_qty = sum(s.quantity for s in skus)
    safety_qty = sum(s.quantity for s in skus if s.assigned_box == "OPT_SAFETY")
    safety_pct = (safety_qty / total_qty) * 100 if total_qty else 0.0

    boxes_wo = [b for b in best["boxes"] if b.box_id != "OPT_SAFETY"]
    vf_without_safety, out_without_safety = opt._evaluate_boxes(boxes_wo)

    rows = []
    for b in best["boxes"]:
        assigned = [s for s in skus if s.assigned_box == b.box_id]
        qty = sum(s.quantity for s in assigned)
        usage = qty / total_qty * 100 if total_qty else 0.0
        vf = (sum(b.calculate_void_fill(s.volume) * s.quantity for s in assigned) / qty) if qty > 0 else None
        rows.append({
            "box_id": b.box_id, "l": b.dim_1, "w": b.dim_2, "h": b.dim_3,
            "Usage (%)": round(usage, 2),
            "Avg_Void_Fill_%": round(vf, 2) if vf is not None else None})
    box_stats = pd.DataFrame(rows)

    print("Best (EXACT 5 boxes, soft-objective)")
    print(f"- Percentile: {best['percentile']}")
    print(f"- Average void fill WITH safety: {vf_with_safety:.2f}%")
    print(f"- Average void fill WITHOUT safety: {vf_without_safety:.2f}%")
    print(f"- Outliers WITH safety: {out_with_safety:.2f}%")
    print(f"- Outliers WITHOUT safety: {out_without_safety:.2f}%")
    print(f"- % items using safety box: {safety_pct:.2f}%")
    print("\nBox range:")
    for b in sorted(best["boxes"], key=lambda x: x.volume):
       print(" ", b)

    print("\nPer-box report:")
    print(box_stats.sort_values("Usage (%)", ascending=False))
```

---

## Conclusion
The `Optimised_boxes_solution.py` script implements a deterministic, percentile-based packaging optimiser. It provides an interpretable and transparent approach to multi-box optimisation without relying on randomisation.


---


# README — Visualisations in the Packaging Performance Dashboard

This document provides a technical explanation of the code and data visualisations implemented in the `Dashboard_comparison.py` script.  
It describes:
- how the **data are prepared and passed on** to the charts,
- how each **visualisation** is built with **Altair** or **Plotly**,  
- and the **purpose of each code block** that generates a chart.  

### How to run the dashboard

When the dashboard is launched, a sidebar appears on the left-hand side of the screen.
Use it to:
- Specify the paths to the **returns dataset, current boxes file, and optimiser script**.
- Check that each file is correctly loaded (a success message appears once detected).
- Click “Run” to execute the analysis.

The visualisations and KPIs will then update automatically based on the selected files.

---

## Part I: Methodological framework and data preparation

### 1.1 Libraries used

The project relies on a selection of Python libraries.

| Library | Main Purpose |
|----------|---------------|
| **Streamlit** | User interface and chart rendering. |
| **Altair** | 2D declarative charts (bar, bubble, pie, boxplot, histogram, scatter). |
| **Plotly** | 3D interactive visualisations. |
| **Pandas** | Data manipulation and preprocessing before plotting. |

---
### 1.2. Data processing
Before any visualisation, the raw data undergo some preparatory stages defined by key functions in the script:

- **Standardisation of dimensions (`prepare_returns`):** The dimensions of each returned item (SKU) are sorted from longest to shortest (`dim_1`, `dim_2`, `dim_3`). This standardisation ensures consistent orientation when evaluating an item's fit within a given box.
- **Box assignment logic (`assign_to_boxes`):** This function simulates the packing process. For each SKU, it iterates through the available boxes (sorted by volume) and assigns the item to the first, and therefore smallest—box that can accommodate it. 
- **KPI calculation (`kpis_calculation`):** This function calculates the performance indicators. The average void fill is calculated as a weighted average, taking into account the quantity of each SKU. This ensures that high-volume items have a proportionally greater impact on the final metric.

### 1.3. Data flow and key variables
The following table describes the main pandas DataFrames created and transformed throughout the script:

| DataFrame | Description | Origin & Purpose |
|------------|--------------|------------------|
| `df_returns` | The initial dataset of returned items. | Loaded from a CSV; serves as the basis for all calculations. |
| `df_boxes_cur` | A list of the boxes currently in use. | Loaded from a CSV; represents the *Current* scenario. |
| `df_boxes_opt` |  The new set of boxes proposed by the algorithm. | Generated by the `evaluate_optimised` function (contained in the external `Optimised_boxes_solution.py` script); this module calculates the optimal set of box dimensions before being loaded into the dashboard for visualisation, and represents the *Optimised* scenario. |
| `df_current_assign` | The result of assigning each SKU to a box in the current scenario. | Output of `assign_to_boxes`; data source for the analysis of the current system. |
| `df_opt_assign` | The result of assigning each SKU to the new set of boxes. | Output of `assign_to_boxes`; data source for the analysis of the optimised system. |
| `per_box_cur` / `per_box_opt_rebuilt` | Aggregated statistics for each box (usage, average void fill). | Generated by `kpis_calculation`; used for summary charts. |
| `outliers` | A subset of SKUs that only fit into the "safety box." | Filtered from `df_opt_assign`; used for the outlier analysis. |

---

## Part II: Analysis of visualisations

This section breaks down each chart in the dashboard, explaining its objective, technical construction, and analytical value.

### 2.1. 3D Scatter Plot: SKU dimensional profile
**Objective:** To provide a three-dimensional representation of the SKU portfolio, positioning each item according to its sorted dimensions, and to capture the diversity and heterogeneity of items in terms of size and shape.  
**Data source:** `df_returns`  
**Technical implementation (Plotly):** A `Scatter3d` chart where the x, y, and z axes correspond to the sorted dimensions. The size of each point is proportional to the quantity, and its colour is also mapped to this quantity.

```python
st.subheader("SKU distribution (3D)")

items = df_returns.copy()
items["Volume_L"] = (items["sku_volume"] / 1_000_000).round(2)
items["dims_txt"] = (
    items["l"].astype(int).astype(str) + " × "
    + items["w"].astype(int).astype(str) + " × "
    + items["h"].astype(int).astype(str) + " mm")

fig_3d = go.Figure(data=[go.Scatter3d(
    x=items["dim_1"], y=items["dim_2"], z=items["dim_3"],
    mode="markers",
    marker=dict(
        size=items["quantity"] / items["quantity"].max() * 20 + 3,
        color=items["quantity"], colorscale="Turbo", showscale=True,
        colorbar=dict(title="Quantity"), line=dict(color="white", width=0.5), opacity=0.7),
    text=items.apply(lambda r: f"SKU: {r['sku_id']}<br>Qty: {r['quantity']:,}<br>Dims: {r['dims_txt']}<br>Vol: {r['Volume_L']:.2f} L", axis=1),
    hovertemplate="%{text}<extra></extra>",)])
```

### 2.2. Bubble chart: Comparison of box volumes
**Objective:** To visually compare the volume of each box between the *Current* and *Optimised* scenarios.   
**Technical implementation (Altair):** A `mark_circle` chart where the x-axis represents the box volume and the y-axis separates the two scenarios.

```python
bubble = (
    alt.Chart(sizes_all)
    .mark_circle(opacity=0.75, stroke="white", strokeWidth=1)
    .encode(
        x=alt.X("Volume_L:Q", title="Volume (L)"),
        y=alt.Y("Scenario:N", title=None, sort=["Optimised", "Current"]),
        size=alt.Size("Volume_L:Q", scale=alt.Scale(range=[80, 1200])),
        colour=alt.Colour("Category:N",
                        scale=alt.Scale(domain=["Current", "Optimised", "Safety"],
                                        range=[COLOR_CURRENT, COLOR_OPTIMISED, COLOR_SAFETY])),
        tooltip=[alt.Tooltip("Scenario:N"), alt.Tooltip("box_id:N"), alt.Tooltip("Volume_L:Q")]))
```

### 2.3. Stacked bar chart: mean void fill by box
**Objective:** To compare the average void fill for each box across the two scenarios.  
**Data source:** `vf_cmp_plot`  
**Technical implementation (Altair):** A `mark_bar` chart where the x-axis represents the box rank (from smallest to largest) and the y-axis shows the mean void fill percentage.

```python
chart_vf_cmp = (
    alt.Chart(vf_cmp_plot)
    .mark_bar()
    .encode(
        x=alt.X("Rank:N", title="Box rank by volume (1 = smallest)"),
        y=alt.Y("Avg_Void_Fill_%:Q", title="Mean void fill (%)"),
        colour=alt.Colour("Scenario:N",
                        scale=alt.Scale(domain=["Current", "Optimised"],
                                        range=[COLOR_CURRENT, COLOR_OPTIMISED])),
        tooltip=[alt.Tooltip("Scenario:N"), alt.Tooltip("box_id:N"), alt.Tooltip("Avg_Void_Fill_%:Q")]))
```

### 2.4. Boxplots: Distribution of void fill
**Objective:** To display the statistical distribution of void fill percentages for each box.  
**Data source:** `boxplot_all`  
**Technical implementation (Altair):** A `mark_boxplot` chart, faceted by scenario. Each boxplot illustrates the median, quartiles, and range (min-max) of void fill for items assigned to that box.

```python
boxplot_chart = (
    alt.Chart(boxplot_all)
    .mark_boxplot(size=50, extent='min-max')
    .encode(
        x=alt.X("Box_Name:N", title="Box ID"),
        y=alt.Y("void_fill_pct:Q", title="Void Fill (%)"),
        colour=alt.Colour("Scenario:N",
                        scale=alt.Scale(domain=["Current", "Optimised"],
                                        range=[COLOR_CURRENT, COLOR_OPTIMISED])),
        column=alt.Column("Scenario:N", title=""),
        tooltip=[alt.Tooltip("selected_box:N"), alt.Tooltip("void_fill_pct:Q", aggregate="median")]))
```

### 2.5. Pie chart: Optimised box usage distribution
**Objective:** To illustrate the utilisation rate of each box in the optimised set.  
**Data source:** `pie_data`  

```python
pie = (alt.Chart(pie_data)
.mark_arc(outerRadius=120, innerRadius=40, stroke="white", strokeWidth=1)
    .encode(
        theta=alt.Theta("Usage (%):Q"),
        colour=alt.Colour("box_id:N", scale=alt.Scale(domain=domain, range=range_colors)),
        tooltip=[alt.Tooltip("box_id:N"), alt.Tooltip("Usage (%):Q")]))
text = (
    alt.Chart(pie_data)
    .mark_text(radius=90, size=11, fontWeight="bold", colour="white")
    .encode(theta=alt.Theta("Usage (%):Q"), text=alt.Text("Usage (%):Q", format=".1f")))
```

### 2.6. Histogram: outlier analysis
**Objective:** To analyse the characteristics of SKUs assigned to the "safety box."  
**Data source:** `outliers`  
- **Histogram:** A `mark_bar` chart that bins outlier SKUs by their volume.
  
```python
hist_volume = (
    alt.Chart(outliers_volumes)
    .mark_bar(colour=COLOR_SAFETY, opacity=0.85)
    .encode(
        x=alt.X("Volume_L:Q", bin=alt.Bin(maxbins=25), title="Volume (Litres)"),
        y=alt.Y("count()", title="Number of SKUs"),
        tooltip=[alt.Tooltip("count()", title="Number of SKUs"),
                 alt.Tooltip("Volume_L:Q", bin=True, title="Volume Range (L)")]))
```

---

## Conclusion

This document provides a methodological description of the visualisation components of the dashboard.
