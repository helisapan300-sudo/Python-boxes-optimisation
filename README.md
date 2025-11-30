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

---

### 4.2 — Bubble Chart (Altair)

**Purpose:** Compare box volumes across Current and Optimized scenarios.

**Data:** `sizes_all`

**Key elements:**
- `.mark_circle()`: draws colored bubbles.
- `x` and `y`: volume vs scenario.
- `size`: bubble area proportional to box volume.
- `color`: scenario or safety box category.
- `tooltip`: displays contextual info (Scenario, Box ID, Dimensions, Volume).

---

### 4.3 — Bar Chart (Mean Void Fill by Volume Rank)

**Purpose:** Compare mean void fill per box ordered by volume.

**Data:** `vf_cmp_plot`

**Key elements:**
- `.mark_bar()`: creates bar chart.
- `x`: rank of box volume.
- `y`: mean void fill (%).
- `color`: scenario color mapping.
- `tooltip`: detailed values on hover.

---

### 4.4 — Boxplots (Void Fill Distribution per Box)

**Purpose:** Show void fill distribution per box for both scenarios.

**Data:** `boxplot_all`

**Key elements:**
- `.mark_boxplot()`: displays median, quartiles, min/max.
- `column`: facets charts side by side (Current / Optimized).
- `color`: distinguishes scenarios.
- `tooltip`: shows detailed statistics for each box.

---

### 4.5 — Pie Chart (Optimized Box Usage)

**Purpose:** Display percentage usage of each optimized box.

**Data:** `pie_data`

**Key elements:**
- `.mark_arc()`: draws donut chart sectors.
- `theta`: sector size based on usage percentage.
- `color`: color-coded per box.
- `tooltip`: shows box ID and usage share.

---

### 4.6 — Histogram (Outlier Volume Distribution)

**Purpose:** Show the volume distribution of SKUs assigned to the safety box.

**Data:** `outliers_volumes`

**Key elements:**
- `.mark_bar()`: bar chart for frequency distribution.
- `bin=alt.Bin(maxbins=25)`: groups data into bins.
- `count()`: counts SKUs per bin.
- `color`: purple tone representing the safety category.

---

### 4.7 — Scatter Plot (Outlier Aspect Ratios)

**Purpose:** Show length/width and length/height ratios of outlier SKUs.

**Data:** `outliers_ratios`

**Key elements:**
- `.mark_circle()`: one point per SKU.
- `x`, `y`: aspect ratios.
- `size`: proportional to SKU volume.
- `tooltip`: provides SKU ID, ratios, and volume.

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

---
