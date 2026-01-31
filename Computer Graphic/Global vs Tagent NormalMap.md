### 1. Why Use This Technique?

In 3D rendering, representing fine surface details (like brick crevices or skin pores) through geometry alone would lead to an explosive increase in vertex counts.

- **Normal Mapping**: This allows us to store complex surface details in a texture and modify pixel normals in real-time, "tricking" the eye into perceiving bumps and depth.
    
- **Benefits of Tangent Space**:
    
    - **Animation Friendly**: Even if the model deforms or rotates (e.g., a character bending their arm), the normals remain correctly oriented relative to the surface.
        
    - **Texture Reusability**: The same normal map can be applied to objects of different shapes.
        
    - **UV Efficiency**: It supports mirrored UV layouts (e.g., the left and right sides of a face sharing the same texture space).
        
    - **Comparison to Global Normals**: Global (Object-space) normals store absolute directions; if the model moves, the lighting breaks immediately because the stored directions no longer match the world orientation.
        

---

### 2. Core Logic and Mathematical Derivation

The goal is to find a local coordinate system at every point on the surface, known as the **TBN Matrix**.

#### A. The Parametric Equation

Assume a point $P$ on a triangle is uniquely determined by UV coordinates $(u, v)$. The relationship between the edge vectors $\vec{E}$, the Tangent vector $\vec{T}$ (aligned with the $u$ direction), and the Bitangent vector $\vec{B}$ (aligned with the $v$ direction) is:

$$\vec{E_1} = \Delta u_1 \vec{T} + \Delta v_1 \vec{B}$$

$$\vec{E_2} = \Delta u_2 \vec{T} + \Delta v_2 \vec{B}$$

#### B. Solving the Matrix Equation

We rewrite these linear equations in matrix form to solve for the unknown $\vec{T}$ and $\vec{B}$:

$$\begin{bmatrix} \vec{E_1} \\ \vec{E_2} \end{bmatrix} = \begin{bmatrix} \Delta u_1 & \Delta v_1 \\ \Delta u_2 & \Delta v_2 \end{bmatrix} \begin{bmatrix} \vec{T} \\ \vec{B} \end{bmatrix}$$

To isolate $\vec{T}$ and $\vec{B}$, we multiply both sides by the **inverse** of the UV matrix:

$$\begin{bmatrix} \vec{T} \\ \vec{B} \end{bmatrix} = \frac{1}{\Delta u_1 \Delta v_2 - \Delta v_1 \Delta u_2} \begin{bmatrix} \Delta v_2 & -\Delta v_1 \\ -\Delta u_2 & \Delta u_1 \end{bmatrix} \begin{bmatrix} \vec{E_1} \\ \vec{E_2} \end{bmatrix}$$

#### C. Building the Darboux Frame (TBN Matrix)

Using the calculated $\vec{T}$ and $\vec{B}$, and the existing interpolated vertex normal $\vec{N}$, we construct the $4 \times 4$ transformation matrix $D$:

$$D = \begin{bmatrix} \text{normalized}(\vec{T}) \\ \text{normalized}(\vec{B}) \\ \text{normalized}(\vec{N}_{interp}) \\ [0, 0, 0, 1] \end{bmatrix}$$

_Note: In the code implementation, $D.transpose()$ is used to transform vectors from Tangent Space back into Model/World Space._

---

### 3. Fragment Shader Logic Summary

1. **Preparation**: Calculate the 3D edge vectors $E$ and the corresponding UV differences $U$ for the triangle.
    
2. **Solve TBN**: Invert $U$ and multiply by $E$ to find the local axes.
    
3. **Texture Sampling**: Use interpolated `uv` coordinates to extract the local normal from `model.normal(uv)`.
    
4. **Space Transformation**: Execute `D.transpose() * normal` to rotate the "relative direction" from the map into an "absolute direction" in the scene.
    
5. **Lighting**: Calculate the `diffuse` intensity by taking the dot product of the final normal $n$ and the light direction $l$.
    

---

### 4. Key Insight: Resolving Your Doubt

#### **The Question: Why do we use an interpolated normal to build the matrix $D$, only to multiply it by the texture normal again later?**

**The Answer:**

Think of this as the relationship between the **"Foundation"** and the **"Decoration."**

1. **Interpolated Normal (The Foundation)**:
    
    This defines the **macro-scale** orientation of the surface. It serves as the $Z$-axis of the TBN coordinate system. Without it, the computer wouldn't know if the surface is facing up, down, left, or right in the big world. It ensures the model looks "smooth" even with low geometry.
    
2. **Texture Normal (The Decoration/Detail)**:
    
    This defines **micro-scale** perturbations. It provides a tiny "offset" relative to the foundation. For example, if the texture says "tilt 5 degrees left," it needs the Foundation (Interpolated Normal) to know which way "left" actually points in 3D space.
    

**The Final Formula:**

$$n_{final} = \text{normalize}(D^T \cdot n_{map})$$

- $n_{map}$: The detailed direction from the texture (usually very close to $(0, 0, 1)$).
    
- $D^T$: Translates this detail, based on the actual "slope" of the triangle (defined by the Tangent and Interpolated Normal), into World Coordinates.
    

**Conclusion**: The interpolated normal is responsible for the **overall smoothness** of the surface, while the normal map is responsible for the **fine-grained roughness**. Together, they allow a low-polygon model to maintain a smooth silhouette while possessing incredibly complex visual textures.