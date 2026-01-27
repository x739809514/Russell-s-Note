## 1. Core Objective

When rendering a triangle on a screen, we must decide two things: which pixels are **inside** the triangle, and what **attributes** (color, depth, texture coordinates) those pixels should have. **Barycentric Coordinates** are the ultimate tool to solve both problems simultaneously.

---

## 2. Core Logic: From 1D to 2D

### 1D Linear Interpolation (Line Segment)

For any point $P$ on a line segment $AB$, its position can be expressed by a ratio $t$:

$$P = (1 - t)A + tB$$

where $t \in [0, 1]$. This is essentially 1D barycentric coordinates, with weights $(1-t)$ and $t$.

### 2D Barycentric Coordinates (Triangle)

For any point $P$ within a triangle $ABC$, we seek three weights $\alpha, \beta$, and $\gamma$ such that:

1. **Position Equation**: $P = \alpha A + \beta B + \gamma C$
    
2. **Normalization Constraint**: $\alpha + \beta + \gamma = 1$
    

**Decision Logic:**

- If $\alpha, \beta, \gamma$ are all $\ge 0$, the point $P$ is **inside** the triangle.
    
- If any coordinate is negative, the point $P$ is **outside** the triangle.
    

---

## 3. Mathematical Implementation: Calculating Weights

### Method A: Area Method (Intuitive)

The barycentric coordinates of point $P$ correspond to the ratio of the area of the sub-triangle formed by $P$ and the opposite edge to the total area:

$$\alpha = \frac{\text{Area}(PBC)}{\text{Area}(ABC)}, \quad \beta = \frac{\text{Area}(PCA)}{\text{Area}(ABC)}, \quad \gamma = \frac{\text{Area}(PAB)}{\text{Area}(ABC)}$$

### Method B: Cross Product (Efficient for Code)

By constructing two vectors $\vec{u}$ and $\vec{v}$ based on the coordinates, we use the cross product to find a vector perpendicular to them, which directly reveals the weights. This is the approach used in the `TinyRenderer` source code.

---

## 4. Rasterization Algorithm Workflow

1. **Find Bounding Box**: Determine the min/max $x, y$ of the three vertices to define a search rectangle.
    
2. **Iterate Pixels**: Loop through every pixel $P(x, y)$ within that bounding box.
    
3. **Calculate Coordinates**: Compute $\alpha, \beta, \gamma$ for the current pixel.
    
4. **Test and Shade**:
    
    - If the coordinates are valid (all $\ge 0$), calculate the point's attributes (like $Z$-buffer depth or color):
        
        $$\text{Attribute}_P = \alpha \cdot \text{Attribute}_A + \beta \cdot \text{Attribute}_B + \gamma \cdot \text{Attribute}_C$$
        
    - Call `set(x, y, color)` to render the pixel.
        

---

## 5. Specific Clarification on "Hard Math"

### ❓ Doubt: The `signed_triangle_area` Formula

**User Concern**:

The formula used in the code `0.5 * ((by-ay)*(bx+ax) + ...)` looks intimidating and the derivation is hard to follow. Is it mandatory to master the derivation?

**Explanation**:

**No, you do not need to memorize the derivation.** Treat this as a standard "Black Box" tool in computational geometry. It is known as the **Shoelace Formula** (or the discrete form of Green's Theorem).

- **How it works**: It calculates the area of trapezoids formed by each edge and the x-axis. As you "walk" around the triangle, the areas added in one direction are subtracted in the other, leaving only the area of the enclosed triangle.
    
- **Why "Signed"?**:
    
    - **Positive result**: The vertices are ordered Counter-Clockwise (CCW).
        
    - **Negative result**: The vertices are ordered Clockwise (CW).
        
- **Practical Advice**: Treat this function as a "plugin" for area and orientation detection. Its importance lies in the output:
    
    1. The absolute value is used as the **denominator** for barycentric weights.
        
    2. The sign is used for **Back-face Culling** (deciding if a triangle is facing toward or away from the camera).