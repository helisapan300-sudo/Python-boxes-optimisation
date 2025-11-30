# README — Visualisations in the Packaging Performance Dashboard

This document explains **how the visualization code** in `Final_dash.py` works.  
It describes:
- how the **data are prepared and passed** to the charts,
- how each **visualization** is built with **Altair** or **Plotly**,  
- and the **purpose of each code block** that generates a chart.  

No business interpretation or analytical commentary is included — this focuses only on the code.

---

## 1. Libraries Used

| Library | Main Purpose |
|----------|---------------|
| **Streamlit** | User interface and chart rendering. |
| **Altair** | 2D declarative charts (bar, bubble, pie, boxplot, histogram, scatter). |
| **Plotly** | 3D interactive visualizations. |
| **Pandas** | Data manipulation and preprocessing before plotting. |

---

## 2. General Visualization Structure

Each visualization follows the same structure:

1. **Data preparation:** Compute or select relevant columns.  
   Example: compute SKU volume or void fill rate.
2. **Chart creation:** Use `alt.Chart()` (Altair) or `go.Figure()` (Plotly) with appropriate encodings.
3. **Customization:** Add colors, titles, legends, axes, and tooltips.
4. **Rendering:** Display the chart in Streamlit with `st.altair_chart()` or `st.plotly_chart()`.

---

## 3. Data Flow Between Sections

| Variable | Description | Used In |
|-----------|--------------|----------|
| `df_returns` | Returns data: `sku_id`, `l`, `w`, `h`, `quantity`. | 3D Scatter |
| `df_boxes_cur` | Current box set. | Bubble Chart, Mean Void Fill |
| `df_boxes_opt` | Optimized box set returned by the optimizer. | Bubble Chart, Mean Void Fill |
| `df_current_assign` | SKU-to-box assignments (current scenario). | Boxplot |
| `df_opt_assign` | SKU-to-box assignments (optimized scenario). | Boxplot, Outliers |
| `per_box_cur` | Box-level statistics (`N Returns`, `Usage (%)`, `Avg_Void_Fill_%`). | Mean Void Fill |
| `per_box_opt_rebuilt` | Same statistics for optimized boxes. | Pie Chart, Mean Void Fill |
| `outliers` | SKUs that did not fit any core box (`selected_box == 'OPT_SAFETY'`). | Histogram, Ratio Scatter |

---

## 4. Visualization Details

### 4.1 — 3D Scatter Plot (Plotly)

**Purpose:** Represent SKU dimensions (`dim_1`, `dim_2`, `dim_3`) in 3D space.

**Data:** `df_returns`

**Key elements:**
- `go.Scatter3d`: defines a 3D scatter plot.
- `x`, `y`, `z`: sorted product dimensions.
- `marker.size`: proportional to quantity.
- `marker.color`: quantity mapped to color (Turbo scale).
- `hovertemplate`: defines hover text.
- `update_layout()`: sets axes, background, and margins.

```python
fig_3d = go.Figure(data=[go.Scatter3d(
    x=items["dim_1"], y=items["dim_2"], z=items["dim_3"],
    mode="markers",
    marker=dict(
        size=items["quantity"] / items["quantity"].max() * 20 + 3,
        color=items["quantity"], colorscale="Turbo", showscale=True,
        colorbar=dict(title="Quantity"), line=dict(color="white", width=0.5), opacity=0.7),
    text=items.apply(lambda r: f"SKU: {r['sku_id']}<br>Qty: {r['quantity']:,}<br>Dims: {r['dims_txt']}<br>Vol: {r['Volume_L']:.2f} L", axis=1),
    hovertemplate="%{text}<extra></extra>",)])
fig_3d.update_layout(height=600, margin=dict(l=0, r=0, b=0, t=30))
st.plotly_chart(fig_3d, use_container_width=True)
```

---

### 4.2 — Bubble Chart (Altair)

**Purpose:** Compare box volumes across Current and Optimized scenarios.

**Data:** `sizes_all`

```python
bubble = (
    alt.Chart(sizes_all)
    .mark_circle(opacity=0.75, stroke="white", strokeWidth=1)
    .encode(
        x=alt.X("Volume_L:Q", title="Volume (L)"),
        y=alt.Y("Scenario:N", title=None, sort=["Optimised", "Current"]),
        size=alt.Size("Volume_L:Q", scale=alt.Scale(range=[80, 1200])),
        color=alt.Color("Category:N",
                        scale=alt.Scale(domain=["Current", "Optimised", "Safety"],
                                        range=[COLOR_CURRENT, COLOR_OPTIMISED, COLOR_SAFETY])),
        tooltip=[alt.Tooltip("Scenario:N"), alt.Tooltip("box_id:N"), alt.Tooltip("Volume_L:Q")]
    )
)
st.altair_chart(bubble, use_container_width=True)
```

---

### 4.3 — Bar Chart (Mean Void Fill by Volume Rank)

**Purpose:** Compare mean void fill per box ordered by volume.

**Data:** `vf_cmp_plot`

```python
chart_vf_cmp = (
    alt.Chart(vf_cmp_plot)
    .mark_bar()
    .encode(
        x=alt.X("Rank:N", title="Box rank by volume (1 = smallest)"),
        y=alt.Y("Avg_Void_Fill_%:Q", title="Mean void fill (%)"),
        color=alt.Color("Scenario:N",
                        scale=alt.Scale(domain=["Current", "Optimised"],
                                        range=[COLOR_CURRENT, COLOR_OPTIMISED])),
        tooltip=[alt.Tooltip("Scenario:N"), alt.Tooltip("box_id:N"), alt.Tooltip("Avg_Void_Fill_%:Q")]
    )
)
st.altair_chart(chart_vf_cmp, use_container_width=True)
```

---

### 4.4 — Boxplots (Void Fill Distribution per Box)

**Purpose:** Show void fill distribution per box for both scenarios.

**Data:** `boxplot_all`

```python
boxplot_chart = (
    alt.Chart(boxplot_all)
    .mark_boxplot(size=50, extent='min-max')
    .encode(
        x=alt.X("Box_Name:N", title="Box ID"),
        y=alt.Y("void_fill_pct:Q", title="Void Fill (%)"),
        color=alt.Color("Scenario:N",
                        scale=alt.Scale(domain=["Current", "Optimised"],
                                        range=[COLOR_CURRENT, COLOR_OPTIMISED])),
        column=alt.Column("Scenario:N", title=""),
        tooltip=[alt.Tooltip("selected_box:N"), alt.Tooltip("void_fill_pct:Q", aggregate="median")]
    )
)
st.altair_chart(boxplot_chart)
```

---

### 4.5 — Pie Chart (Optimized Box Usage)

**Purpose:** Display percentage usage of each optimized box.

**Data:** `pie_data`

```python
pie = (
    alt.Chart(pie_data)
    .mark_arc(outerRadius=120, innerRadius=40, stroke="white", strokeWidth=1)
    .encode(
        theta=alt.Theta("Usage (%):Q"),
        color=alt.Color("box_id:N", scale=alt.Scale(domain=domain, range=range_colors)),
        tooltip=[alt.Tooltip("box_id:N"), alt.Tooltip("Usage (%):Q")]
    )
)
text = (
    alt.Chart(pie_data)
    .mark_text(radius=90, size=11, fontWeight="bold", color="white")
    .encode(theta=alt.Theta("Usage (%):Q"), text=alt.Text("Usage (%):Q", format=".1f"))
)
st.altair_chart(pie + text, use_container_width=True)
```

---

### 4.6 — Histogram (Outlier Volume Distribution)

**Purpose:** Show the volume distribution of SKUs assigned to the safety box.

**Data:** `outliers_volumes`

```python
hist_volume = (
    alt.Chart(outliers_volumes)
    .mark_bar(color=COLOR_SAFETY, opacity=0.85)
    .encode(
        x=alt.X("Volume_L:Q", bin=alt.Bin(maxbins=25), title="Volume (Litres)"),
        y=alt.Y("count()", title="Number of SKUs"),
        tooltip=[alt.Tooltip("count()", title="Number of SKUs"),
                 alt.Tooltip("Volume_L:Q", bin=True, title="Volume Range (L)")]
    )
)
st.altair_chart(hist_volume, use_container_width=True)
```

---

### 4.7 — Scatter Plot (Outlier Aspect Ratios)

**Purpose:** Show length/width and length/height ratios of outlier SKUs.

**Data:** `outliers_ratios`

```python
ratio_chart = (
    alt.Chart(outliers_ratios)
    .mark_circle(size=80, opacity=0.7, color=COLOR_SAFETY)
    .encode(
        x=alt.X("Ratio_L/W:Q", title="Length/Width Ratio", scale=alt.Scale(domain=[0, 10])),
        y=alt.Y("Ratio_L/H:Q", title="Length/Height Ratio", scale=alt.Scale(domain=[0, 10])),
        size=alt.Size("Volume_L:Q", title="Volume (L)", scale=alt.Scale(range=[50, 400])),
        tooltip=[alt.Tooltip("sku_id:N"), alt.Tooltip("Ratio_L/W:Q"), alt.Tooltip("Ratio_L/H:Q")]
    )
)
ref_line = alt.Chart(pd.DataFrame({'x': [0, 10], 'y': [0, 10]})).mark_line(strokeDash=[5, 5], color='gray').encode(x='x:Q', y='y:Q')
st.altair_chart(ratio_chart + ref_line, use_container_width=True)
```

---

## 5. Streamlit Integration

Charts are displayed in Streamlit using:
- `st.altair_chart()` for Altair charts,
- `st.plotly_chart()` for Plotly figures,
- `st.divider()` to visually separate dashboard sections.

Section titles and layout:
- `st.subheader()`: section headings,
- `st.caption()`: minor text or notes,
- `st.columns()`: layout for side-by-side KPIs.

---

## 6. Summary

| Step | Action | Library |
|-------|---------|----------|
| Data preparation | Cleaning, computation, aggregation | Pandas |
| Chart creation | Define chart type and encodings | Altair / Plotly |
| Customization | Colors, scales, legends, tooltips | Altair / Plotly |
| Rendering | Integrate and display in dashboard | Streamlit |

Each visualization block is independent and designed to **display processed data cleanly** without using HTML.  
Colors and styling are handled natively by the visualization libraries.
