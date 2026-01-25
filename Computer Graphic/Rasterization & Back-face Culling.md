### 1. Core Objective: Triangle Filling

In 2D screen space, given three vertices $A, B, C$, our goal is to determine which pixels reside within the boundaries of the triangle and color them accordingly.

### 2. Algorithmic Evolution

The author guides us through a progression from "naive" to "industry standard" approaches:

- **Line Sweeping (The Old Way):** Dividing the triangle into a top and bottom half and tracing the boundaries row by row. It is complex to implement and prone to "gaps" between triangles.
    
- **Bounding Box (The Modern Way):** 1. Define a bounding box using the minimum and maximum coordinates of the vertices: `xmin, xmax, ymin, ymax`.
    
    2. Iterate through every pixel $(x, y)$ within this box.
    
    3. Check if the point lies inside the triangle using Barycentric coordinates.
    

### 3. Mathematical Core: Barycentric Coordinates

To determine if a point $P$ is inside triangle $ABC$, we express $P$ as a linear combination of the vertices:

$$P = (1 - u - v)A + uB + vC$$

Where $u$ and $v$ are scalar coefficients.

- **Condition:** A point $P$ is inside the triangle if and only if $u \ge 0$, $v \ge 0$, and $(1 - u - v) \ge 0$.
    
- **Implementation:** In the code, this is solved by finding the cross product of vectors to see how "aligned" the point is with the triangle's edges.
    

---

### 4. Key Logic: Back-face Culling

This addresses your specific area of confusion regarding why we filter triangles based on their "area."

#### Query: Why does "Area" determine the orientation of a triangle?

**Explanation and Principle:**

In 3D modeling, the order in which vertices are defined (Winding Order) determines the direction of the surface **normal**. When projected onto a 2D screen, this order appears either **Clockwise (CW)** or **Counter-Clockwise (CCW)**.

We use the **Signed Area** to detect this. For vertices $(x_0, y_0), (x_1, y_1), (x_2, y_2)$, the area is derived from the cross product of two edge vectors:

$$Area = \frac{1}{2} \left| \vec{AB} \times \vec{AC} \right|$$

In 2D plane coordinates, this expands to:

$$2 \cdot Area = (x_1 - x_0)(y_2 - y_0) - (y_1 - y_0)(x_2 - x_0)$$

- **Logic Breakdown:**
    
    - If $Area > 0$: The vertices are CCW. The triangle is facing the camera. **Keep it.**
        
    - If $Area < 0$: The vertices are CW. The triangle is facing away from the camera. **Discard it.**
        
    - If $Area \approx 0$: The triangle is degenerate (a line or a point). **Discard it.**
        

**Code Logic:**

C++

```
// Calculate twice the signed area
float total_area = (B.x - A.x) * (C.y - A.y) - (B.y - A.y) * (C.x - A.x);

// If area is less than 1 pixel (includes negative area), return
if (total_area < 1) return; 
```

> **Note:** The author uses `< 1` instead of `< 0` to perform a double optimization: it culls back-faces (negative values) and also skips "sub-pixel" triangles that are too small to meaningfully contribute to the image.

---

### 5. Remaining Challenges

While back-face culling removes "hidden" back surfaces, it cannot handle **Occlusion** (i.e., when two front-facing triangles overlap).

- **The Artifact:** You might see the back of the mouth or teeth rendered "on top" of the lips because they are both front-facing, but the teeth should be blocked by the lips.
    
- **The Solution:** This requires the **Z-buffer (Depth Buffer)** algorithm, which is the focus of the next lesson.