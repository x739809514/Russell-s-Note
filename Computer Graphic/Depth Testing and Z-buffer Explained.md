## 1. Why Use This Technology?

In 3D rendering, objects have spatial occlusion (overlapping) relationships. If we simply draw triangles in the order they appear in the model file, the triangles rendered later will always overwrite the earlier ones. This results in a "visual mess" where back-facing parts (like a model's leg) appear in front of the torso.

To solve the **Hidden Surface Removal** problem, several methods were historically used:

- **Back-face Culling**: Only removes surfaces facing away from the camera; it cannot handle cases where an object's front part obscures its own back part or where multiple objects overlap.
    
- **Painter's Algorithm**: Sorts triangles by depth and draws them from far to near. However, it cannot handle "cyclic overlap" (where A blocks B, B blocks C, and C blocks A) and the sorting overhead per frame is massive.
    

**Z-buffer technology** solves all these issues perfectly by performing depth comparisons at the **pixel level**. It is a cornerstone of modern computer graphics.

## 2. Core Ideas and Logic

### A. Shift from Geometry Sorting to Pixel Competition

Instead of caring about the order of triangles, we maintain a "depth profile" for every single pixel $(x, y)$ on the screen.

### B. Two Buffers

1. **Frame Buffer**: Stores the color values ($RGB$) of the pixels.
    
2. **Z-buffer (Depth Buffer)**: Stores the depth value $z$ of the closest object currently known for that pixel.
    

### C. Rendering Logic Flow

For every triangle to be rendered:

1. **Iterate Pixels**: Find all pixels $P(x, y)$ covered by the triangle (using its bounding box).
    
2. **Calculate Depth**: Use Barycentric coordinates (see below) to calculate the exact 3D depth value $z_{current}$ at that specific point.
    
3. **Depth Test (Z-Test)**:
    
    - If $z_{current} > z_{buffer}[x][y]$ (assuming larger $z$ is closer):
        
        - This means the current pixel is closer to the viewer than whatever was drawn there before.
            
        - **Update**: Set $z_{buffer}[x][y] = z_{current}$ and write the color to the Frame Buffer.
            
    - Otherwise: Discard the pixel; do nothing.
        

## 3. Mathematical Derivation: Depth Interpolation

To find the depth of any point inside a triangle during the rasterization stage, we use **Barycentric Coordinates**.

Given a triangle with vertices $A, B, C$ and their respective depth values $z_A, z_B, z_C$ in 3D space. For any point $P$ inside the triangle, its depth $z_P$ is calculated as:

$$z_P = \alpha z_A + \beta z_B + \gamma z_C$$

Where $\alpha, \beta, \gamma$ are the Barycentric coordinates of point $P$, satisfying:

$$\alpha + \beta + \gamma = 1$$

$$\alpha, \beta, \gamma \ge 0$$

Using this interpolation ensures that the depth across the surface of the triangle is smooth and continuous.

## 4. Technical Comparison: Why Z-buffer is Better

|**Feature**|**Painter's Algorithm**|**Z-buffer**|
|---|---|---|
|**Sorting Target**|Entire Triangles|Individual Pixels|
|**Complexity**|High ($O(n \log n)$ for sorting)|Low (Linear per pixel/triangle)|
|**Memory Cost**|Minimal|Higher (Requires a depth map)|
|**Cyclic Occlusion**|Fails to handle|Handles perfectly|
|**Parallelization**|Difficult|Very easy (Hardware friendly)|

---

## 5. Doubt Summary & Clarification

**Doubt: In the TinyRenderer code or demo, why does a larger Z value represent a distance closer to the camera?**

**Answer:**

This depends entirely on **Coordinate System Conventions** and the design of the **Mapping Function**:

1. **Coordinate Convention**: The author typically uses a right-handed coordinate system where the positive $Z$-axis points out of the screen toward the viewer. Thus, a larger numerical value is physically closer to you.
    
2. **Code Mapping**: In the `project()` function, the author maps the original model coordinates (usually between $[-1, 1]$) to a screen space of $[0, 255]$.
    
    - Mapping formula: $z_{screen} = (z_{model} + 1.0) \times \frac{255.0}{2.0}$
        
    - When the model point $z = -1$ (far), the screen $z = 0$.
        
    - When the model point $z = 1$ (near), the screen $z = 255$.
        
3. **Logical Consistency**: As long as you initialize the Z-buffer with the minimum value (like 0) and use the comparison `if (current_z > buffer_z)`, the system remains logically sound. This is simply a choice made by the author to make the code feel more intuitive (larger = better/closer).