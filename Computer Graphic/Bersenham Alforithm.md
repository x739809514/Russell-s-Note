## 📐 Summary: Bresenham's Line Algorithm Evolution

### 1. The Parametric Approach (The Starting Point)

The simplest way to define a line between points $A(a_x, a_y)$ and $B(b_x, b_y)$ is using a parameter $t \in [0, 1]$:

$$\begin{cases} x(t) = a_x + t(b_x - a_x) \\ y(t) = a_y + t(b_y - a_y) \end{cases}$$

**Problem:** Choosing a fixed step for $t$ causes gaps if the line is longer than the number of samples.

---

### 2. Modern Sampling & Symmetry (The "Bulletproof" Version)

To avoid gaps, we iterate over $x$ (the "independent" variable) and calculate $y$. If the line is "steep" ($|dy| > |dx|$), we swap $x$ and $y$ to ensure we always iterate along the longest axis.

- **Equation:** $y = a_y + (x - a_x) \cdot \frac{dy}{dx}$
    

> **你的提问与解答：**
> 
> **问：** 为什么代码里要进行 `swap` 操作？
> 
> **答：** 如果直线太“陡”（y轴的变化比x轴大），只按x轴步进会导致点太稀疏，线会断掉。通过交换坐标，我们将“陡线”转为“平线”处理，画完后再转回来，保证了线条连续。

---

### 3. Optimization Round 1: Incremental Error

Instead of re-calculating $y$ from scratch using $y = mx + c$, we use the fact that each step $\Delta x = 1$ results in a constant change in $y$ equal to the slope $m = \frac{dy}{dx}$.

- **Update rule:** $y_{i+1} = y_i + \frac{dy}{dx}$
    

> **你的提问与解答：**
> 
> **问：** Optimization Round 1 具体是怎么优化的？
> 
> **答：** 它把“乘法”变成了“加法”。利用斜率固定的原理，x每走1步，y就累加一个固定的偏移量（斜率），避免了循环内复杂的比例运算。

---

### 4. Optimization Round 2: Decision Variable

To prepare for integer-only math, we track a "derror" (the distance between the actual line and the current pixel center).

- **Logic:** If $\text{error} > 0.5$, then increment $y$ and $\text{error} \leftarrow \text{error} - 1.0$.
    

> **你的提问与解答：**
> 
> **问：** `error -= 1.0` 是为什么？
> 
> **答：** 当误差累积超过0.5个像素时，我们将像素坐标 $y$ 向上移了一格。这一移位相当于把“基准线”抬高了1.0，所以相对误差要减去1.0。

---

### 5. Optimization Round 3: Integer-Only Math

To eliminate floating-point precision issues and speed up calculation, we multiply the entire inequality by $2 \cdot dx$.

- **Original:** $\frac{dy}{dx} > 0.5$
    
- **Integer Form:** $2 \cdot dy > dx$
    
- **New error update:** $\text{error} \leftarrow \text{error} + 2 \cdot dy$
    
- **New threshold:** If $\text{error} > dx$, then $y \leftarrow y + 1$ and $\text{error} \leftarrow \text{error} - 2 \cdot dx$.
    

> **你的提问与解答：**
> 
> **问：** Round 3 还是没懂，为什么要把小数消灭？
> 
> **答：** 整数运算比浮点数快得多。通过给等式两边同时乘以分母，我们将原本需要处理的“0.5”和“斜率”变成了纯整数加减法。

---

### 6. Optimization Round 4: Removing Branches

Modern CPUs use branch prediction. Frequent "if" statements in a tight loop can cause pipeline stalls. Round 4 uses bitwise/arithmetic tricks to remove the `if` branch.

- **Branchless update:** $y += (slope\_sign) \cdot (error > threshold)$
    

> **你的提问与解答：**
> 
> **问：** Round 4 那个“丑陋”的优化是什么意思？
> 
> **答：** 它利用布尔表达式（真=1，假=0）参与算术运算。通过 `y += (1) * (error > dx)` 这种写法，强制让CPU不进行分支跳转，从而在海量绘图时达到最高性能。

---

### 7. Application: OBJ Rendering (The Homework)

Rendering a 3D model requires parsing `.obj` files and mapping 3D coordinates to a 2D viewport.

- **Viewport Mapping:** $x_{screen} = \frac{(x_{obj} + 1) \cdot \text{width}}{2}$
    

> **你的提问与解答：**
> 
> **问：** `.obj` 文件里的 `v_indices` 是存什么的？解析脚本怎么写？
> 
> **答：** `v_indices` 存的是三角形顶点的“编号”。解析时，先用 `vector` 存下所有 `v` 开头的坐标，读到 `f`时，根据编号回表查坐标，最后调用 `line` 函数连线。
