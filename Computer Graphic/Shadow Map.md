### 一、 核心概念总结

#### 1. Shadow Mapping 的本质

阴影映射是一种基于**深度对比**的算法。它的核心逻辑非常直白：

- **光源视角的真相**：从光源出发，在这个方向上能撞击到的第一个物体就是“亮的”。
    
- **相机视角的对比**：对于相机看到的每一个点，我们计算它到光源的距离。如果这个距离比光源记录的“最近距离”要远，说明它被挡住了，它就在阴影里。
    

#### 2. “两次渲染”流程

- **第一遍（Shadow Pass）**：把相机放在光源位置，只记录深度信息（Z-buffer），生成 **Shadow Map**。
    
- **第二遍（Camera Pass）**：从正常相机位置渲染模型，对每个像素进行“阴影测试”。
    

#### 3. 坐标系的“穿越”与“撤销”

这是你最困惑的部分。我们渲染时在**相机屏幕空间**，但对比阴影必须在**光源空间**。

- **为什么需要“撤销”？** 因为相机视角下的 $(x, y)$ 是屏幕坐标，$z$ 是相对相机的深度。光源根本不认这些数据。
    
- **撤销到哪里？** 我们利用相机矩阵的逆矩阵 $(Viewport \cdot P \cdot V)^{-1}$，把像素点“推回”到**世界空间**。
    
- **再投射**：回到世界空间后，再乘以光源的 $M_{light}$ 矩阵，将其转换到光源的屏幕坐标。
    
- **总结**：世界空间是连接相机和光源的“翻译官”。
    

---

### 二、 关键技术细节

1. **Shadow Acne（阴影粉刺）与 Bias**：由于浮点数精度问题，物体表面会认为自己遮挡了自己。我们通过在深度对比时引入一个微小的偏移量 `bias` 来解决。
    
2. **BlankShader 的作用**：它是一个极其简化的着色器，不计算颜色和光照，只负责在第一遍渲染时填满光源视角的 Z-buffer。
    
3. **Z-buffer 的可视化 (`drop_zbuffer`)**：为了调试方便，我们将内存中的深度数据映射到 $0 \sim 255$ 的灰度值，生成 TGA 图片。其中 `z < -100` 的过滤是为了剔除背景，使模型深度变化更清晰。
    
4. **深度值的含义**：在 TinyRenderer 中，$z$ 值越大通常表示离镜头越近。因此判定逻辑是：如果`当前点深度 < 光源记录的最近深度`，则说明当前点更远，处于阴影中。
    

---

### 三、 代码逻辑深度解析

基于你提供的 `main.cpp` 实现，我们将逻辑拆解为后期处理模式：

#### 1. 矩阵的准备

C++

```
// M: 将相机屏幕坐标 (x,y,z) 还原回世界坐标
mat<4,4> M = (Viewport * Perspective * ModelView).invert(); 

// N: 将世界坐标投射到光源的屏幕坐标
// 注意：此时 ModelView 已经是 lookat(light, ...) 之后的状态
mat<4,4> N = Viewport * Perspective * ModelView; 
```

#### 2. 后期处理循环（Shadow Test）

C++

```
for (int x=0; x<width; x++) {
    for (int y=0; y<height; y++) {
        // 步骤 A: 找到当前像素在相机眼里的位置
        double z = zbuffer_copy[x+y*width];
        
        // 步骤 B: 坐标穿越
        // 1. (x, y, z) -> M -> 世界坐标 (fragment)
        vec4 fragment = M * vec4{x, y, z, 1.}; 
        
        // 2. 世界坐标 -> N -> 光源屏幕坐标 (q)
        vec4 q = N * fragment;
        vec3 p = q.xyz() / q.w; // 归一化，得到 p.x, p.y (查表坐标) 和 p.z (实际深度)

        // 步骤 C: 深度比对 (关键判定)
        // zbuffer 是光源视角下的深度图
        // 如果 p.z（当前点）比 zbuffer 里存的（最近点）还要小，说明被挡住了
        bool lit = (p.z > zbuffer[int(p.x) + int(p.y)*shadoww] - .03);
        
        mask[x+y*width] = lit;
    }
}
```

#### 3. 颜色应用

C++

```
if (!mask[idx]) { // 如果不亮 (即在阴影里)
    TGAColor c = framebuffer.get(x, y);
    // 简单的调暗处理：将颜色分量乘以一个系数
    framebuffer.set(x, y, TGAColor(c[0]*0.3, c[1]*0.3, c[2]*0.3, 255));
}
```


# The Processing for ShadowMap
## 1. Core Logic: The "Two-Pass" Strategy

Shadow Mapping is a technique used to determine which parts of a scene are in shadow by viewing the scene from the light's perspective. It relies on a simple principle: **If a point is visible from the light's position, it is lit; if it is blocked by another object, it is in shadow.**

To implement this, we perform two rendering passes:

### Pass 1: The Shadow Pass (Generating the Shadow Map)

1. **Switch Perspective**: We move the "camera" to the light's position using `lookat(light, center, up)`.
    
2. **Record Depth**: We render the scene using a simplified shader (like `BlankShader`). We don't care about colors or textures here; we only want to record the distance from the light to the nearest object in every direction.
    
3. **The Result**: This distance data is stored in a `zbuffer` (the **Shadow Map**).
    

### Pass 2: The Camera Pass (Final Rendering)

1. **Return to Camera**: We reset the perspective to the actual camera using `lookat(eye, center, up)`.
    
2. **Render Colors**: We render the final image as usual (e.g., using `PhongShader`).
    
3. **Shadow Testing**: For every pixel we draw, we check if it is "hidden" from the light by comparing its distance to the light against the value stored in our Shadow Map.
    

---

## 2. The Transformation Matrices ($M$ and $N$)

The most technical part is "translating" coordinates between the camera's view and the light's view.

- **Matrix $M$ (The Undo Button)**: This is the inverse of the camera's combined transformation matrix: $M = (Viewport_C \cdot Projection_C \cdot ModelView_C)^{-1}$. It takes a pixel from your **Camera Screen** and pushes it back into **3D World Space**.
    
- **Matrix $N$ (The Light's View)**: This is the light's transformation matrix: $N = Viewport_L \cdot Projection_L \cdot ModelView_L$. It takes a point from **3D World Space** and projects it onto the **Light's Screen** (the Shadow Map).
    

**Why "Undo" to World Space?** Because the camera and the light speak different "coordinate languages." World space is the common ground where they can both agree on where an object is located.

---

## 3. Code Implementation & Explanation

### The `BlankShader`

This shader is the "surveyor." It calculates where things are but doesn't "paint" them.

- **Vertex Shader**: It applies the Light's MVP to find the screen position from the light's POV.
    
- **Fragment Shader**: It returns `false` (do not discard) and a dummy color. The real work happens automatically when `rasterize` updates the `zbuffer` with the $z$ value.
    

### The Post-Processing Loop

This is where the "Shadow Test" happens for every pixel $(x, y)$ in the final image:

C++

```
// 1. Back-project: From Camera Screen -> World Space
vec4 fragment = M * vec4{x, y, zbuffer_copy[x+y*width], 1.};

// 2. Re-project: From World Space -> Light's Screen
vec4 q = N * fragment;
vec3 p = q.xyz() / q.w; 

// 3. Compare Depths:
// p.z is the ACTUAL distance from the light to this point.
// zbuffer[...] is the SHORTEST distance the light could see.
bool lit = (p.z > zbuffer[int(p.x) + int(p.y)*shadoww] - .03); 
```

- **`p.z > zbuffer[...] - .03`**: If the actual distance is greater than the recorded distance (minus a small bias to prevent "Shadow Acne" artifacts), it means something else is closer to the light. Therefore, the point is in shadow.
    

---

## 4. Troubleshooting: Why is the model missing?

If your `shadowmap.tga` or `framebuffer.tga` are empty:

1. **Check `lookat` parameters**: Ensure `center` is actually where your model is. If your model is at $(0,0,0)$ but your `center` is $(10,10,10)$, you are looking at empty space.
    
2. **Check Model Scale**: If your model is very small (0.01 units) or very large (10,000 units), it might be invisible or clipped.
    
3. **Check Viewport**: Ensure `init_viewport` covers the actual image size.
    
4. **Visualize the Depth**: Use `drop_zbuffer` to check `zbuffer2.tga`. If you see a silhouette, your first pass is correct, and the issue lies in the transformation matrices $M$ or $N$.