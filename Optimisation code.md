# README: Technical Guide to the Packaging Optimisation Engine

## Introduction
This document provides a detailed technical explanation of the `Optimised_boxes_solution.py` script. The script defines a percentile-based optimisation engine that determines an optimal set of five box dimensions. The objective is to minimise void fill while limiting the number of outlier SKUs that would require an oversized "safety box."

---

## Part I: Methodological Framework

### 1.1. Object-Oriented Modelling
The core of the script is structured around three main classes that model the problem domain:

- **`SKU`**: Represents a single stock-keeping unit. Each instance holds its dimensions (systematically sorted from largest to smallest for orientation-independent fit assessment), volume, and quantity. It also tracks which box it has been assigned to.

- **`Box`**: Represents a type of packaging box. Similar to `SKU`, its dimensions are sorted. It includes essential methods to check if a SKU can fit inside (`can_fit`) and to calculate the resulting void fill percentage (`calculate_void_fill`).

- **`BoxOptimiser`**: The central class that orchestrates the optimisation process. It takes a list of SKU objects as input and manages the generation, evaluation, and selection of candidate box configurations.

### 1.2. Optimisation Approach
The algorithm implements a **grid search** over a defined set of percentiles. It does not use stochastic methods (like genetic algorithms or simulated annealing) but instead follows a deterministic, data-driven approach.

The process is as follows:
1. **Candidate Generation**: For each percentile value in a predefined grid (e.g., 86, 88, ..., 94), a unique five-box configuration is generated.
2. **Evaluation**: Each configuration is evaluated by simulating the assignment of all SKUs. This evaluation yields two key metrics: the average void fill (weighted by quantity) and the rate of outliers (items that fit in none of the five boxes).
3. **Selection**: The "best" configuration is selected using an objective function that combines these two metrics.

---

## Part II: Detailed Algorithmic Logic

### 2.1. Box Configuration Generation (`_create_boxes`)
This method is central to the creation of candidate solutions. The process is as follows:

- **Data Weighting**: First, the SKU dataset is weighted by quantity. This means the dimensions of high-volume items are repeated proportionally, ensuring their influence on the statistical distribution is accurate.

- **Volume Partitioning**: The weighted SKUs are then divided into five groups (quantiles) based on their volume. Each group thereby represents a slice of the item volume distribution.

- **Percentile-based Dimension Calculation**: For each group, the dimensions of the corresponding box are determined by calculating a specified percentile (e.g., the 90th percentile) of the dimensions (`dim_1`, `dim_2`, `dim_3`) of the SKUs within that group. Using a percentile (rather than the maximum value) is a heuristic designed to create boxes that fit the majority of items in a group, while accepting that a few exceptionally large items may become outliers.

- **Operational Rounding**: The calculated dimensions are rounded up to the nearest 10mm for practical feasibility and standardisation.

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

### 2.2. Evaluation and Objective Function (`_evaluate_boxes`, `_objective`)
Once a five-box configuration is created, it must be evaluated:

- **Assignment**: Each SKU is assigned to the smallest available box (sorted by volume) into which it can fit. If it fits in none, it is marked as an outlier.

- **Metric Calculation**: The script calculates the weighted average void fill for all assigned items and the percentage of outliers relative to the total item quantity.

- **Objective Function**: The performance of the configuration is calculated with the following score:

  Score = Void Fill (%) + lambda x Outlier Rate (%) 

  The λ (lambda) coefficient is a hyperparameter that serves as a penalty factor. A high λ value heavily penalises configurations that produce many outliers, thus steering the optimisation toward more inclusive solutions at the cost of a potentially higher void fill.

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

### 2.3. Outlier Management (`_add_safety_box_to`)
If the selected configuration produces outliers, an optional "safety box" can be added. Its dimensions are calculated to be just large enough to accommodate the largest of the outlier items, with a small margin. The evaluation is then re-run with this additional box to obtain final metrics that reflect a 100% fulfilment solution.

```python
def _add_safety_box_to(self, boxes: List[Box]) -> None:
    outliers = [s for s in self.skus if s.assigned_box is None]
    if not outliers:
        return
    d1 = int(math.ceil((max(s.dim_1 for s in outliers)+20)/50)*50)
    d2 = int(math.ceil((max(s.dim_2 for s in outliers)+20)/50)*50)
    d3 = int(math.ceil((max(s.dim_3 for s in outliers)+20)/50)*50)
    boxes.append(Box("OPT_SAFETY", d1, d2, d3))
```

---

## Part III: Outputs and Reporting

The main `optimise_packaging()` function executes the full process and generates the following outputs:

- **Console Summary**: Prints a summary of key results, including average void fill and outlier rate both *with* and *without* the safety box.
- **Box Dimensions**: Displays the dimensions of each optimised box.
- **Per-Box Statistics**: A pandas DataFrame containing, for each box, its usage rate and average void fill. This DataFrame is structured for direct use in the Streamlit visualisation dashboard.

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
The `Optimised_boxes_solution.py` script implements a deterministic, percentile-based packaging optimiser. It provides an interpretable and transparent approach to multi-box optimisation without relying on randomisation. Through clear object-oriented structure and robust reporting, it serves as the computational foundation for the Streamlit visualisation dashboard.

