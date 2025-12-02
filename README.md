# README — Beginner-Friendly Explanation of the Packaging Dashboard (`Final_dash.py`)

This document explains the **Streamlit dashboard** step by step, in **clear British English**.  
It focuses on how the code works technically and how each visualisation is built.  
You can read this file on GitHub or export it to PDF.

---

## 1) What the Dashboard Does

The dashboard compares two packaging scenarios:
- **Current**: the boxes in use now,
- **Optimised**: a new set of boxes produced by the optimisation script.

It shows **KPIs** (key performance indicators) and **charts** to help you understand:
- How well items fit (void fill),
- How many outliers there are,
- How box sizes and usage compare between scenarios,
- Which items need the safety box and why.

---

## 2) How to Run the Dashboard

### Install Dependencies
```bash
pip install streamlit pandas altair plotly
```

### Start the App
```bash
streamlit run Final_dash.py
```

### Configure File Paths in the Sidebar
At the top of `Final_dash.py` (after imports), the app shows a sidebar where you can point to your files:

```python
with st.sidebar:
    st.header("Configuration")
    returns_file = st.text_input("Returns CSV", os.path.join(BASE_DIR, "returns_data.csv"))
    current_boxes_file = st.text_input("Current boxes CSV", os.path.join(BASE_DIR, "current_boxes.csv"))
    optimiser_file = st.text_input("Optimiser script (.py)", os.path.join(BASE_DIR, "Optimised_boxes_solution.py"))
    run_btn = st.button("Run", type="primary")

if not run_btn:
    st.info("Set file paths and run.")
    st.stop()
```

> Make sure you have `import os` and a `BASE_DIR` defined (e.g. `BASE_DIR = os.path.dirname(__file__)`).

### Data Loading Guard
Before loading data, the app checks the files exist:
```python
for p in [returns_file, current_boxes_file, optimiser_file]:
    if not os.path.exists(p):
        st.error(f"File not found: {p}")
        st.stop()
```

---

## 3) Core Libraries Used

| Library | Why it’s used |
|--------|---------------|
| **Streamlit** (`st`) | Builds the web app (sections, metrics, layout, charts). |
| **Pandas** (`pd`) | Loads CSVs, cleans and transforms data for charts. |
| **Altair** (`alt`) | Creates elegant 2D charts (bar, boxplot, scatter, pie/donut). |
| **Plotly** (`go`) | Creates interactive 3D scatter plots. |

---

## 4) Streamlit Building Blocks (used throughout)

- `st.subheader("...")` — section titles.
- `st.columns(n)` — layout with multiple columns side by side.
- `with st.container(border=True):` — visual box around content (helps grouping KPIs).
- `st.metric(label, value, delta=..., delta_color=...)` — KPI cards.
- `st.altair_chart(chart, use_container_width=True)` — display Altair charts.
- `st.plotly_chart(fig, use_container_width=True)` — display Plotly figures.
- `st.divider()` — visual separator between sections.
- `st.caption("...")` and `st.markdown("...")` — explanatory text.

> **Tip:** Streamlit runs top-to-bottom on every interaction. Keep heavy computations cached if needed (e.g., with `@st.cache_data`).

---

## 5) KPIs Section — Current vs Optimised

The dashboard shows quick metrics in two columns, such as **Average Void Fill** and **Outlier Rate**.

### How the layout works
```python
st.markdown("**Average void fill**")
v1, v2 = st.columns(2)

with v1:
    with st.container(border=True):
        # Current KPI card
        st.metric("Void fill", f"{cur_vf:.2f}%", label_visibility="collapsed")

with v2:
    with st.container(border=True):
        # Optimised KPI card with delta
        st.metric(
            "Void fill",
            f"{opt_vf:.2f}%",
            delta=f"{(opt_vf - cur_vf):+.2f} pp",
            delta_color="inverse",
            label_visibility="collapsed"
        )
```
- `st.columns(2)` creates two side-by-side areas.
- `st.container(border=True)` draws a box around each KPI.
- `st.metric` shows a value and optional **delta** (change in percentage points).  
  `delta_color="inverse"` flips the colour logic (e.g. lower is better).

The **Outlier Rate** uses the same structure, but the labels/variables point to outliers.

---

## 6) Chart 1 — 3D Scatter: “SKU distribution (3D)”

**Purpose:** show the distribution of items in 3D space using their **sorted dimensions**.  
**Data:** `df_returns` (after preparing columns like `dim_1`, `dim_2`, `dim_3`, `quantity`).

### Preparation
```python
items = df_returns.copy()
items["Volume_L"] = (items["sku_volume"] / 1_000_000).round(2)
items["dims_txt"] = (
    items["l"].astype(int).astype(str) + " × " +
    items["w"].astype(int).astype(str) + " × " +
    items["h"].astype(int).astype(str) + " mm")
```
- Compute volume in **litres** for tooltips.
- Create a human-readable **dimensions string** for hover text.

### Build the Plotly graph
```python
fig_3d = go.Figure(data=[go.Scatter3d(
    x=items["dim_1"], y=items["dim_2"], z=items["dim_3"],
    mode="markers",
    marker=dict(
        size=items["quantity"] / items["quantity"].max() * 20 + 3,
        color=items["quantity"],
        colorscale="Turbo",
        showscale=True,
        colorbar=dict(title="Quantity"),
        line=dict(color="white", width=0.5),
        opacity=0.7
    ),
    text=items.apply(lambda r: f"SKU: {r['sku_id']}<br>Qty: {r['quantity']:,}<br>Dims: {r['dims_txt']}<br>Vol: {r['Volume_L']:.2f} L", axis=1),
    hovertemplate="%{text}<extra></extra>",
)])

fig_3d.update_layout(
    scene=dict(
        xaxis=dict(title="Longest (mm)", backgroundcolor="rgb(230,230,230)", gridcolor="white"),
        yaxis=dict(title="Second (mm)",  backgroundcolor="rgb(230,230,230)", gridcolor="white"),
        zaxis=dict(title="Smallest (mm)", backgroundcolor="rgb(230,230,230)", gridcolor="white"),
    ),
    height=600, margin=dict(l=0, r=0, b=0, t=30),
)

st.plotly_chart(fig_3d, use_container_width=True)
```
- `Scatter3d` plots each SKU as a point in 3D.
- Marker **size** is scaled by quantity; **colour** maps to quantity with a colour bar.
- `hovertemplate` shows custom rich text.

---

## 7) Chart 2 — Bubble Chart: “Box sizes” (Altair)

**Purpose:** compare **volumes** of boxes between **Current** and **Optimised** scenarios; highlight **Safety** category.

### Prepare the data
```python
cur_sizes = df_boxes_cur_sorted.copy()
cur_sizes["Scenario"] = "Current"
cur_sizes["Volume_L"] = cur_sizes["box_volume"] / 1_000_000
cur_sizes["dims"] = (
    cur_sizes["l"].astype(int).astype(str) + " × " +
    cur_sizes["w"].astype(int).astype(str) + " × " +
    cur_sizes["h"].astype(int).astype(str) + " mm")
cur_sizes = cur_sizes[["Scenario", "box_id", "Volume_L", "dims"]]

opt_sizes = df_boxes_opt_sorted.copy()
opt_sizes["box_volume"] = opt_sizes["l"] * opt_sizes["w"] * opt_sizes["h"]
opt_sizes["Scenario"] = "Optimised"
opt_sizes["Volume_L"] = opt_sizes["box_volume"] / 1_000_000
opt_sizes["dims"] = (
    opt_sizes["l"].astype(int).astype(str) + " × " +
    opt_sizes["w"].astype(int).astype(str) + " × " +
    opt_sizes["h"].astype(int).astype(str) + " mm")
opt_sizes = opt_sizes[["Scenario", "box_id", "Volume_L", "dims"]]

sizes_all = pd.concat([cur_sizes, opt_sizes], ignore_index=True)
sizes_all["Category"] = sizes_all.apply(lambda r: "Safety" if "SAFETY" in str(r["box_id"]) else r["Scenario"], axis=1)
```
- Build a **harmonised** dataset for both scenarios.
- Add a `Category` column so that *Safety* can be coloured separately.

### Build the chart
```python
bubble = (
    alt.Chart(sizes_all)
    .mark_circle(opacity=0.75, stroke="white", strokeWidth=1)
    .encode(
        x=alt.X("Volume_L:Q", title="Volume (L)"),
        y=alt.Y("Scenario:N", title=None, sort=["Optimised", "Current"]),
        size=alt.Size("Volume_L:Q", title="Volume (L)", scale=alt.Scale(range=[80, 1200])),
        color=alt.Color("Category:N",
                        scale=alt.Scale(domain=["Current", "Optimised", "Safety"],
                                        range=[COLOR_CURRENT, COLOR_OPTIMISED, COLOR_SAFETY]),
                        legend=alt.Legend(title=None, orient="top", direction="horizontal")),
        tooltip=[
            alt.Tooltip("Scenario:N"),
            alt.Tooltip("box_id:N", title="Box ID"),
            alt.Tooltip("dims:N", title="Dimensions"),
            alt.Tooltip("Volume_L:Q", title="Volume (L)", format=".2f"),
        ],
    )
    .properties(height=200)
)
st.altair_chart(bubble, use_container_width=True)
```
- Size and x-position both encode **volume**, making the comparison easy to see.
- Colour indicates **Current**, **Optimised**, or **Safety**.

---

## 8) Chart 3 — Bar Chart: “Mean void fill by box (ranked by volume)”

**Purpose:** compare **average void fill** per box across scenarios, ordered by **box rank** (1 = smallest by volume).

### Prepare the data
```python
cur_rank = df_boxes_cur_sorted.copy()
cur_rank = cur_rank.assign(Rank=cur_rank["box_volume"].rank(method="first").astype(int))[["box_id", "Rank"]]

opt_rank = df_boxes_opt_sorted.copy()
opt_rank = opt_rank.assign(box_volume=opt_rank["l"] * opt_rank["w"] * opt_rank["h"])
opt_rank = opt_rank.assign(Rank=opt_rank["box_volume"].rank(method="first").astype(int))[["box_id", "Rank"]]

cur_vf = per_box_cur.merge(cur_rank, on="box_id", how="left").assign(Scenario="Current")
opt_vf = per_box_opt_rebuilt.merge(opt_rank, on="box_id", how="left").assign(Scenario="Optimised")

df_vf_cmp = pd.concat([
    cur_vf[cur_vf["box_id"] != "OPT_SAFETY"],
    opt_vf[opt_vf["box_id"] != "OPT_SAFETY"]
], ignore_index=True)
```
- Generate a **rank** based on box volume for each scenario.
- Combine both scenarios into one DataFrame.

### Build the chart
```python
vf_cmp_plot = df_vf_cmp.dropna(subset=["Rank", "Avg_Void_Fill_%"]).copy()
vf_cmp_plot["Rank"] = vf_cmp_plot["Rank"].astype(int).astype(str)

chart_vf_cmp = (
    alt.Chart(vf_cmp_plot)
    .mark_bar()
    .encode(
        x=alt.X("Rank:N", title="Box rank by volume (1 = smallest)",
                sort=alt.SortField(field="Rank", order="ascending")),
        y=alt.Y("Avg_Void_Fill_%:Q", title="Mean void fill (%)"),
        color=alt.Color("Scenario:N",
                        scale=alt.Scale(domain=["Current", "Optimised"],
                                        range=[COLOR_CURRENT, COLOR_OPTIMISED]),
                        legend=alt.Legend(orient="bottom")),
        tooltip=[
            alt.Tooltip("Scenario:N"),
            alt.Tooltip("box_id:N", title="Box ID"),
            alt.Tooltip("Rank:N", title="Rank"),
            alt.Tooltip("Avg_Void_Fill_%:Q", title="Mean void fill (%)", format=".2f"),
        ],
    )
    .properties(height=320)
)
st.altair_chart(chart_vf_cmp, use_container_width=True)
```
- Bars grouped by **rank**; colours separate **Current** and **Optimised**.

---

## 9) Chart 4 — Boxplot: “Void Fill Distribution per Box (excluding safety)”

**Purpose:** show the **distribution** (median, quartiles, min-max) of void fill for each box, **split by scenario**.

### Prepare the data
```python
cur_boxplot_data = df_current_assign[
    (df_current_assign["selected_box"].notna()) &
    (df_current_assign["void_fill_rate"].notna())
].copy()
cur_boxplot_data["void_fill_pct"] = cur_boxplot_data["void_fill_rate"] * 100
cur_boxplot_data["Scenario"] = "Current"

opt_boxplot_data = df_opt_assign[
    (df_opt_assign["selected_box"].notna()) &
    (df_opt_assign["selected_box"] != "OPT_SAFETY") &
    (df_opt_assign["void_fill_rate"].notna())
].copy()
opt_boxplot_data["void_fill_pct"] = opt_boxplot_data["void_fill_rate"] * 100
opt_boxplot_data["Scenario"] = "Optimised"

boxplot_all = pd.concat([cur_boxplot_data, opt_boxplot_data], ignore_index=True)
```
- Filter out missing values and the safety box for clean, fair comparison.
- Convert **void fill** to percentage.

### Attach rank for ordering and build chart
```python
_rank_merge = pd.concat([
    cur_rank.assign(Scenario="Current"),
    opt_rank.assign(Scenario="Optimised")
], ignore_index=True)

boxplot_all["selected_box"] = boxplot_all["selected_box"].astype(str)
_rank_merge["box_id"] = _rank_merge["box_id"].astype(str)

boxplot_all = boxplot_all.merge(
    _rank_merge, left_on=["selected_box", "Scenario"], right_on=["box_id", "Scenario"], how="left"
)
boxplot_all["Box_Name"] = boxplot_all["selected_box"]
boxplot_all["Sort_Key"] = boxplot_all["Rank"].astype(str).str.zfill(2)

boxplot_chart = (
    alt.Chart(boxplot_all)
    .mark_boxplot(size=50, extent='min-max')
    .encode(
        x=alt.X(
            "Box_Name:N",
            title="Box ID",
            sort=alt.EncodingSortField(field="Sort_Key", order="ascending")
        ),
        y=alt.Y("void_fill_pct:Q", title="Void Fill (%)", scale=alt.Scale(domain=[0, 100])),
        color=alt.Color(
            "Scenario:N",
            scale=alt.Scale(domain=["Current", "Optimised"], range=[COLOR_CURRENT, COLOR_OPTIMISED]),
            legend=alt.Legend(title="Scenario", orient="top", direction="horizontal"),
        ),
        column=alt.Column("Scenario:N", title="", sort=["Current", "Optimised"]),
        tooltip=[
            alt.Tooltip("selected_box:N", title="Box ID"),
            alt.Tooltip("Scenario:N"),
            alt.Tooltip("Rank:O", title="Rank"),
            alt.Tooltip("void_fill_pct:Q", title="Median Void Fill (%)", aggregate="median", format=".1f"),
        ],
    )
    .properties(width=350, height=400)
    .configure_view(strokeWidth=0)
    .configure_axis(gridColor='#f0f0f0', domainColor='#999')
    .resolve_scale(x='independent')
)
st.altair_chart(boxplot_chart, use_container_width=False)
```
- A **facet** (split) by scenario lets you compare distributions side by side.

---

## 10) Chart 5 — Pie/Donut: “Box usage (optimised)”

**Purpose:** show how often each **optimised** box is used.

### Prepare and plot
```python
pie_data = per_box_opt_rebuilt.copy()
pie_data = pie_data[~pie_data["Usage (%)"].isna()].sort_values("box_id")

base_colors = ["#FF6B6B", "#FFB347", "#6BCB77", "#4D96FF", "#00C9A7", "#FF8FAB", "#FFD93D", "#9D4EDD"]
domain = list(pie_data["box_id"])
range_colors = [COLOR_SAFETY if "SAFETY" in b else base_colors[i % len(base_colors)] for i, b in enumerate(domain)]

pie = (
    alt.Chart(pie_data)
    .mark_arc(outerRadius=120, innerRadius=40, stroke="white", strokeWidth=1)
    .encode(
        theta=alt.Theta("Usage (%):Q", stack=True),
        color=alt.Color("box_id:N", scale=alt.Scale(domain=domain, range=range_colors),
                        legend=alt.Legend(title="Box", orient="right")),
        tooltip=[alt.Tooltip("box_id:N", title="Box ID"),
                 alt.Tooltip("Usage (%):Q", title="Usage (%)", format=".2f")],
    )
)
text = (
    alt.Chart(pie_data)
    .mark_text(radius=90, size=11, fontWeight="bold", color="white")
    .encode(theta=alt.Theta("Usage (%):Q", stack=True),
            text=alt.Text("Usage (%):Q", format=".1f"),
            order=alt.Order("box_id:N"))
)
st.altair_chart((pie + text).properties(height=400, width=400, title="Usage across optimised boxes"), use_container_width=True)
```
- Donut chart (`innerRadius`) displays **share of usage** per box.
- Labels placed inside the slices via `mark_text`.

---

## 11) Chart 6 — Outliers Analysis (Histogram + Scatter of ratios)

This section appears only if there are **outliers** that require the `OPT_SAFETY` box.

### Summary cards
The code computes statistics: total outlier quantity, share of total, number of distinct SKUs, safety box size and volume, and its average void fill.

### Histogram: outlier volumes
```python
outliers_volumes = outliers.copy()
outliers_volumes["Volume_L"] = (outliers_volumes["sku_volume"] / 1_000_000).round(2)

hist_volume = (
    alt.Chart(outliers_volumes)
    .mark_bar(color=COLOR_SAFETY, opacity=0.85)
    .encode(
        alt.X("Volume_L:Q", bin=alt.Bin(maxbins=25), title="Volume (Litres)"),
        alt.Y("count()", title="Number of SKUs"),
        tooltip=[
            alt.Tooltip("count()", title="Number of SKUs"),
            alt.Tooltip("Volume_L:Q", bin=True, title="Volume Range (L)", format=".2f")
        ]
    )
    .properties(height=350)
)
st.altair_chart(hist_volume, use_container_width=True)
```
- Bins outlier SKUs by **volume** to show their spread.

### Scatter of aspect ratios
```python
outliers_ratios = outliers_volumes.copy()
outliers_ratios["Ratio_L/W"] = outliers_ratios["dim_1"] / outliers_ratios["dim_2"]
outliers_ratios["Ratio_L/H"] = outliers_ratios["dim_1"] / outliers_ratios["dim_3"]
outliers_ratios["Ratio_W/H"] = outliers_ratios["dim_2"] / outliers_ratios["dim_3"]

ratio_chart = (
    alt.Chart(outliers_ratios)
    .mark_circle(size=80, opacity=0.7, color=COLOR_SAFETY)
    .encode(
        x=alt.X("Ratio_L/W:Q", title="Length/Width Ratio", scale=alt.Scale(domain=[0, 10])),
        y=alt.Y("Ratio_L/H:Q", title="Length/Height Ratio", scale=alt.Scale(domain=[0, 10])),
        size=alt.Size("Volume_L:Q", title="Volume (L)", scale=alt.Scale(range=[50, 400])),
        tooltip=[
            alt.Tooltip("sku_id:N", title="SKU"),
            alt.Tooltip("Ratio_L/W:Q", title="L/W Ratio", format=".2f"),
            alt.Tooltip("Ratio_L/H:Q", title="L/H Ratio", format=".2f"),
            alt.Tooltip("Volume_L:Q", title="Volume (L)", format=".2f"),
        ]
    )
    .properties(height=400)
)

ref_line = alt.Chart(pd.DataFrame({'x': [0, 10], 'y': [0, 10]})).mark_line(
    strokeDash=[5, 5], color='gray'
).encode(x='x:Q', y='y:Q')

st.altair_chart(ratio_chart + ref_line, use_container_width=True)
st.caption("Items far from the diagonal (1:1) have extreme proportions (long & thin or flat & wide).")
```
- Shows if outliers are **extreme shapes** (far from 1:1 ratios).

---

## 12) Good Practices and Tips

- **Consistent dimensions:** make sure all lengths are in mm throughout.
- **Avoiding missing data:** drop or fill `NaN`s before charting to prevent empty plots.
- **Performance:** cache heavy calculations with `@st.cache_data` if processing large datasets.
- **Colours:** keep the same colours for Current / Optimised / Safety for visual consistency.
- **Reproducibility:** control random seeds if you later add sampling (none is used here).
