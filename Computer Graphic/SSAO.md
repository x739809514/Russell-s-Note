## Part 1: Knowledge Summary from TinyRenderer SSAO

The document explains the evolution from basic ambient lighting to a more realistic **Screen-Space Ambient Occlusion (SSAO)** implementation.

### 1. Indirect Lighting and Occlusion

- In the standard Phong model, ambient light is treated as a constant, which is unrealistic because it ignores how geometry blocks light.
    
- **Ambient Occlusion (AO)** represents the degree to which a point on a surface is occluded by surrounding geometry, preventing it from receiving indirect light.
    
- Deep crevices and inner corners exhibit higher occlusion, making them appear darker in the final image.
    

### 2. From Brute-Force to Screen-Space

- **Brute-Force AO**: This involves rendering hundreds of shadow maps from various directions and averaging them. While accurate, it is too slow for real-time applications.
    
- **SSAO**: Instead of querying the full 3D scene, it uses the pre-rendered **Depth Buffer (Z-buffer)**.
    
- The computational cost of SSAO depends on screen resolution rather than scene complexity, making it highly efficient.
    

### 3. Core SSAO Logic

- For every pixel (fragment) on the screen, the algorithm takes several random samples in its neighborhood.
    
- It compares the actual depth of these sample points in the 3D scene against the depth stored in the Z-buffer.
    
- If a sample point is "behind" the geometry stored in the Z-buffer, it is considered occluded.
    
- The final ambient intensity is determined by the ratio of occluded to non-occluded samples.
    

---

## Part 2: Consolidated Discussion & Code Explanation

### 1. Algorithm Flow

As illustrated in your provided lecture slide, the process follows these steps:

- **Pixel Iteration**: For each pixel, emit rays outwards in UV (screen) space.
    
- **Depth Sampling**: Along a short path, record the "maximum slope" encountered by comparing depth values.
    
- **Occlusion Check**: A slope of $0$ indicates an open area, while a slope approaching $90$ indicates heavy occlusion.
    
- **Final Shading**: The ambient value is inversely proportional to the local "hilly-ness" or geometric density.
    

### 2. Implementation Breakdown

**A. Initialization and Parameters**

C++

```
constexpr double ao_radius = .1;  // SSAO sampling radius in Normalized Device Coordinates
constexpr int nsamples = 128;     // Number of samples taken per pixel
```

- **`ao_radius`**: Defines the search area for occluders. A larger radius creates softer, more spread-out shadows, while a smaller radius captures tight details.
    
- **`nsamples`**: Determines the quality. More samples reduce noise but increase the performance cost.
    

**B. Smoothing with Smoothstep**

C++

```
auto smoothstep = [](double edge0, double edge1, double x) {
    double t = std::clamp((x - edge0)/(edge1 - edge0), 0., 1.);
    return t*t*(3 - 2*t); // Hermite interpolation
};
```

- **Purpose**: This function creates a smooth $S$-shaped transition for the occlusion factor. It prevents harsh "pop-in" artifacts by gradually fading the occlusion effect as samples approach the edge of the radius.
    

**C. Correcting the Narrowing Conversion Error**

We addressed the C++ error **"invalid narrowing conversion from 'double' to 'unsigned char'"** found in your code:

C++

```
// Corrected application of the SSAO factor to the framebuffer
framebuffer.set(x, y, {
    (unsigned char)(c[0] * ssao), // Explicitly cast the double result back to char
    (unsigned char)(c[1] * ssao), 
    (unsigned char)(c[2] * ssao), 
    c[3]                          // Keep Alpha channel as is
});
```

- **The Fix**: Since `ssao` is a `double` between $0.0$ and $1.0$, multiplying it by a color byte results in a `double`. C++ brace initialization `{}` strictly forbids implicit conversion to a smaller type (`unsigned char`), so an explicit cast is required.