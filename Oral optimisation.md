# README — Beginner-Friendly Technical Explanation of the Packaging Optimisation Engine

This document explains the `Optimised_boxes_solution.py` script step by step.  
It is written in clear, beginner-friendly British English.  
The aim is to help readers understand *how* each function works and *why* each line of code is used.  

---

## 1. Purpose of the Script

This Python program finds an **optimal set of five box sizes** to package a variety of products (SKUs — Stock Keeping Units).  
It tries to reduce the **empty space** inside boxes (called *void fill*) while also making sure that almost every item fits in one of those boxes.

The algorithm tests several box configurations, evaluates each one, and selects the best-performing set.

---

## 2. How the Script Works Overall

The script follows this logical order:

1. **Read SKU data** (items with their length, width, height, and quantity).  
2. **Generate candidate box sizes** based on SKU statistics (using percentiles).  
3. **Evaluate** how well each box set performs (void fill and outlier rate).  
4. **Score** each configuration with an *objective function*.  
5. **Choose** the configuration with the best score (lowest value).  
6. **Optionally**, create a *safety box* large enough to fit any leftover items (outliers).

---

## 3. Function-by-Function Explanation

Below are detailed explanations of the main functions in the optimiser.

---

### 3.1 `_create_boxes(self, n_boxes: int, percentile: int)`

**Purpose:**  
Creates a list of boxes by analysing the SKUs. It calculates the box dimensions based on a chosen percentile (for example, 90th percentile).

This means that the algorithm doesn’t take the *largest item* in a group, but one that covers 90% of them — a practical compromise.

**How it works step by step:**

```python
weighted = [[s.dim_1, s.dim_2, s.dim_3, s.volume]
            for s in self.skus
            for _ in range(s.quantity)]
```
- This is called a **list comprehension**.  
- It repeats each SKU’s dimensions as many times as its quantity.  
  Example: if one SKU has quantity = 3, it appears three times.  
- This gives more importance to frequently sold items.

```python
df = pd.DataFrame(weighted, columns=["dim_1", "dim_2", "dim_3", "volume"])
```
- Converts the weighted list into a **DataFrame** using `pandas`.  
- Makes it easier to calculate statistical values like percentiles.

```python
df["group"] = pd.qcut(df["volume"], q=n_boxes, labels=False, duplicates="drop")
```
- `pd.qcut()` divides the SKUs into groups of equal *frequency* (number of items).  
- Each group corresponds to one future box size.

```python
if df["group"].nunique() < n_boxes:
    df["group"] = pd.cut(df["volume"], bins=n_boxes, labels=False, include_lowest=True)
```
- Sometimes, `qcut` merges bins if the data is not varied enough.  
- This line fixes that by forcing exactly `n_boxes` bins (using `pd.cut()`).

Now for each group, the function calculates the box dimensions:

```python
for gid in sorted(df["group"].unique()):
    g = df[df["group"] == gid] if not df[df["group"] == gid].empty else df
    dim1 = g["dim_1"].quantile(percentile / 100.0)
    dim2 = g["dim_2"].quantile(percentile / 100.0)
    dim3 = g["dim_3"].quantile(percentile / 100.0)
```
- For every group, it calculates the **percentile** (e.g. 90th) for each dimension.  
- This means 90% of items in that group will fit in the box.

Next, we make the values more realistic:

```python
dim1 = max(int(math.ceil(dim1 / 10.0) * 10), 200)
dim2 = max(int(math.ceil(dim2 / 10.0) * 10), 100)
dim3 = max(int(math.ceil(dim3 / 10.0) * 10), 50)
```
- `math.ceil()` rounds numbers *up* to the nearest 10 millimetres.  
- `max()` ensures no box is smaller than a practical minimum.  
  (for example, never smaller than 200×100×50 mm)

Finally, create a Box object:

```python
boxes.append(Box(f"OPT_{len(boxes)+1}", dim1, dim2, dim3))
```
- Adds each box to the list, giving it an ID like `OPT_1`, `OPT_2`, etc.

**End of function:**  
The function returns a list of exactly 5 `Box` objects.

---

### 3.2 `_evaluate_boxes(self, boxes: List[Box])`

**Purpose:**  
Simulates placing all SKUs into boxes and calculates how efficient the configuration is.

**Step by step:**

```python
ordered = sorted(boxes, key=lambda b: b.volume)
```
- Sorts the boxes by their *volume* (from smallest to largest).  
- Ensures we always try to fit items in the smallest possible box first.

```python
for s in self.skus:
    s.assigned_box = None
    dims = s.get_dimensions()
    for b in ordered:
        if b.can_fit(dims):
            s.assigned_box = b.box_id
            break
```
- Loops through every SKU.  
- Resets its assignment.  
- Checks each box in order — the first one that fits is assigned.  
- If no box fits, the SKU remains an “outlier”.

```python
total = sum(s.quantity for s in self.skus)
outliers = sum(s.quantity for s in self.skus if s.assigned_box is None)
```
- Counts the total number of items and how many could not be packed.

```python
vf_sum = 0.0
fit_q = 0
for s in self.skus:
    if s.assigned_box:
        b = next(bb for bb in boxes if bb.box_id == s.assigned_box)
        vf_sum += b.calculate_void_fill(s.volume) * s.quantity
        fit_q += s.quantity
avg_vf = (vf_sum / fit_q) if fit_q else 100.0
```
- Calculates **average void fill** — how much empty space remains in boxes.  
- Weighted by quantity: large orders have more influence.

```python
out_rate = (outliers / total) * 100 if total else 0.0
return avg_vf, out_rate
```
- Returns two key numbers: average void fill (%) and outlier rate (%).

---

### 3.3 `_objective(self, void_fill_pct, outlier_rate_pct)`

**Purpose:**  
Combines the two performance measures into a single score.

```python
return void_fill_pct + self.lambda_outliers * outlier_rate_pct
```
- The variable `lambda_outliers` acts as a *penalty multiplier*.  
- If this value is large, the optimiser strongly avoids outliers (safe but less efficient).  
- If small, it prioritises compact boxes even if some items don’t fit.

This score allows the program to easily *compare* different box configurations.

---

### 3.4 `_optimise(self)`

**Purpose:**  
Runs the full optimisation process — tests multiple configurations and picks the best.

**Detailed steps:**

```python
candidates = []
for p in self.percentile_grid:
    boxes = self._create_boxes(n_boxes=5, percentile=p)
    vf, out = self._evaluate_boxes(boxes)
```
- For each percentile (e.g., 86, 88, 90…), create 5 boxes and test them.

```python
if self.add_safety_box and out > 0:
    self._add_safety_box_to(boxes)
    vf, out = self._evaluate_boxes(boxes)
```
- If `add_safety_box` is `True`, a large backup box is added for any unassigned SKUs.

```python
score = self._objective(vf, out)
candidates.append({
    "percentile": p,
    "void_fill": vf,
    "outlier_rate": out,
    "score": score,
    "boxes": boxes,
})
```
- Calculates the configuration’s score and stores the result.

```python
best = min(candidates, key=lambda x: x["score"])
vf, out = self._evaluate_boxes(best["boxes"])
best.update({"void_fill": vf, "outlier_rate": out})
return best
```
- Finds the configuration with the **lowest score** (best performance).  
- Returns a dictionary containing all important data about the best set of boxes.

---

### 3.5 `_add_safety_box_to(self, boxes)`

**Purpose:**  
Adds a “safety” box large enough to fit *any* item that didn’t fit earlier.

```python
outliers = [s for s in self.skus if s.assigned_box is None]
if not outliers:
    return
```
- Creates a list of unassigned SKUs.  
- If there are none, stops the function early.

```python
d1 = int(math.ceil(max(s.dim_1 for s in outliers)/50)*50)
d2 = int(math.ceil(max(s.dim_2 for s in outliers)/50)*50)
d3 = int(math.ceil(max(s.dim_3 for s in outliers)/50)*50)
boxes.append(Box("OPT_SAFETY", d1, d2, d3))
```
- Finds the **largest** dimension among all outliers for each side.  
- Rounds them up to the nearest 50 mm to give a small buffer.  
- Creates a new `Box` called `"OPT_SAFETY"` and adds it to the list.

This ensures every SKU has at least one box it can fit into.

---

## 4. Key Programming Concepts Used

| Concept | Description |
|----------|--------------|
| **Functions** | Blocks of code that perform a specific task. |
| **Classes and Objects** | Used to model SKUs and Boxes as real-world entities. |
| **List Comprehensions** | Short syntax for creating lists from loops. |
| **If Statements** | Used to check conditions (e.g. if a box can fit an item). |
| **Loops (for)** | Used to go through items, boxes, and configurations. |
| **Sorting and Filtering** | Sorting ensures the smallest box is tried first. |
| **Percentiles** | Measure used to find representative values in data. |

---

## 5. How to Run the Script

To use the optimisation program, you can run it from your terminal:

```bash
python Optimised_boxes_solution.py
```
You will then be asked to provide the name of a CSV file that contains your SKU data.

**Required CSV format:**  
```
sku_id, l, w, h, quantity
```

Example:
```
A001, 250, 180, 70, 10
A002, 320, 210, 110, 5
A003, 150, 100, 80, 20
```

The script will print the best set of boxes and performance results in your console.

---

## 6. Summary

- The algorithm uses **percentiles** to propose practical box sizes.  
- It measures **void fill** and **outlier rate** to find the balance between efficiency and inclusiveness.  
- It uses a simple **objective function** to pick the best combination automatically.  
- A **safety box** ensures that no item is left unassigned.  

By reading and understanding this file, you should now be able to follow the logic of the optimisation program step by step.

---
