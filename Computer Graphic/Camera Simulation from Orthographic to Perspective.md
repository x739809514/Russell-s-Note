### 1. Core Technology: Vertex Transformation

The logical core of this chapter is: **Instead of moving the camera, we rotate the world.** In a renderer, we achieve view switching by modifying the model's vertex coordinates while keeping the "camera" conceptually fixed.

#### Rotation Logic

To rotate the model around the $y$-axis, we apply a rotation matrix $R_y$ to each vertex $v$:

$$R_y = \begin{bmatrix} \cos\alpha & 0 & \sin\alpha \\ 0 & 1 & 0 \\ -\sin\alpha & 0 & \cos\alpha \end{bmatrix}$$

$$v_{rotated} = Ry \cdot v_{original}$$

- **Why use matrices:** Matrices compress complex rotations, scaling, and translations into a unified multiplication operation, which is ideal for hardware acceleration (the essence of a GPU is a massive matrix math engine).
    
- **Advantages:** Compared to manually calculating trigonometric functions for every coordinate, matrix multiplication is more systematic and supports "transformation nesting" (e.g., rotating then translating can be done by simply multiplying two matrices).

#### Perspective Projection Logic

Prior to this, the renderer used "orthographic projection," where objects remained the same size regardless of distance. To simulate the "foreshortening" effect of the human eye, **perspective division** is introduced.

Assume the camera is at $z = c$ and the screen is at $z = 0$. Based on the principle of similar triangles, the transformed coordinates $(x', y')$ are:

$$x' = \frac{x}{1 - z/c}, \quad y' = \frac{y}{1 - z/c}$$

- **Why use this technology:** It provides the image with a sense of "depth" and realism.
    
- **Advantages:** Compared to complex Ray Casting, performing simple division at the vertex stage is computationally inexpensive and serves as the cornerstone of real-time rendering (game engines).

### 2. Rendering Pipeline Improvements

1. **Read Model Vertices.**
    
2. **Rotation Transformation:** Change the orientation of the object in space.
    
3. **Perspective Transformation:** Adjust vertex coordinates to create near-large/far-small distortion.
    
4. **Viewport Transformation:** Map the $[-1, 1]$ space to screen pixel coordinates (e.g., $800 \times 800$).
    
5. **Rasterization:** Fill triangles and use the Z-buffer to handle occlusion.

---

## FAQ & Insights

### Q: Why does multiplying every vertex by a rotation matrix complete the rotation for the entire model?

**Answer:**

This is based on the properties of **Linear Algebra**. In 3D rendering, a model is composed of triangles, which are defined by their vertices.

- **Shape Preservation:** A rotation matrix is an "isometric transformation." It moves the vertices while ensuring that the relative distances and angles between them remain constant.
    
- **Linear Interpolation:** Once you rotate the three vertices of a triangle, the rasterizer interpolates between these new positions. Therefore, not only do the points move, but the faces, textures, and lighting tied to those points rotate correctly as well.
    
- **Efficiency:** This "vertices-represent-the-surface" approach avoids the need to calculate every single internal pixel; you only need to process a small amount of vertex data.

---

## Legacy Issue: The Diablo "Clipped Hand" Phenomenon

### Symptom

When rendering the Diablo model using the logic above, the hand closest to the camera appears broken or "clipped."

### Root Cause

- **Precision and Overflow:** For educational simplicity, the text stores the Z-buffer as an **8-bit grayscale image** (range $0 \sim 255$).
    
- **Calculation Chain:** After rotation and perspective transformation, some $z$-coordinates exceed the original $[-1, 1]$ range. When mapped to the $0 \sim 255$ integer space, an **integer overflow** occurs (e.g., a value of 256 becomes 0 in an 8-bit container).
    
- **Logical Failure:** The overflow causes a point that is actually at the "very front" to be assigned a depth value representing the "very back," leading the depth test to incorrectly discard or overwrite it.

### Solution

- **Technology:** Use a **Floating-point Z-buffer**.
    
- **Approach:** Stop using `TGAImage` to store depth. Instead, use a `std::vector<float>` to store the raw $z$ values.
    
- **Advantages:** Floating-point numbers provide a massive range and high precision, fundamentally eliminating rendering holes caused by overflow.