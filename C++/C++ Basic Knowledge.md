## 🚀 C++ Concepts Summary

### 1. Keywords & Core Syntax

- **`constexpr`**: Tells the compiler an expression can be evaluated at **compile-time**. It moves work from the user's computer to your computer (during compilation).
    
- **`const` vs. `constexpr`**: `const` means "read-only" at runtime; `constexpr` means "constant" at compile-time.
    
- **`unsigned char`**: An 8-bit integer ($0–255$). The "gold standard" for storing pixel data (RGBA) and raw bytes.
    
- **`#define`**: A preprocessor "search and replace" tool. In `stb_image`, `STB_IMAGE_IMPLEMENTATION` acts as a **switch** to turn the header into a source file.
    

### 2. Memory & Pointers

- **Stack vs. Heap**:
    
    - **Stack**: Fast, automatic (e.g., `vector2 p = vector2(0,0)`). Use this for small, short-lived objects.
        
    - **Heap**: Large, manual (using `new`), slower. Requires careful management.
        
- **Pointer to Pointer (`char**`)**: A memory address that points to another address. Often used for arrays of strings (like `argv` in `main`).
    
- **Smart Pointers**:
    
    - `std::unique_ptr`: Sole ownership (default choice).
        
    - `std::shared_ptr`: Shared ownership via reference counting.
        
    - `std::weak_ptr`: Observes a `shared_ptr` without owning it (prevents memory leaks).
        

### 3. Object-Oriented Programming (OOP)

- **Operator `::` vs `.`**: `::` accesses a **Scope** (class/namespace), while `.` accesses a specific **Object's** members.
    
- **`const` after a function**: `int n() const;` means the function is "read-only" and won't change any class variables.
    
- **Forward Declaration**: Writing `class TGAImage;` instead of `#include` to break circular dependencies and speed up compilation.
    
- **Rule of Zero**: In modern C++, if you use smart pointers and vectors, you rarely need to write a manual destructor (`~ClassName`).
    

### 4. C++ Casting (The "Big Four")

|**Cast**|**Use Case**|
|---|---|
|**`static_cast`**|Safe, basic conversions (e.g., `int` to `float`).|
|**`dynamic_cast`**|Safe navigation in inheritance (requires virtual functions).|
|**`const_cast`**|Adding or removing `const` (use with extreme caution!).|
|**`reinterpret_cast`**|Raw bit-level reinterpretation (dangerous).|

---

## 🛠 Standard Library (STL) Toolbox

### 1. Containers & Tables

- **`std::vector`**: The go-to dynamic array.
    
- **`std::unordered_map`**: The C++ version of a **Dictionary**. High performance ($O(1)$) using a hash table.
    
- **`std::map`**: A dictionary that keeps keys sorted ($O(\log n)$).
    

### 2. Streams & Parsing

- **`std::ifstream`**: Opens a file for reading.
    
- **`std::istringstream`**: Treats a string like a file—perfect for parsing `.obj` files line by line.
    
- **`.eof()` Trap**: Don't use `while(!in.eof())`; instead, use `while(in >> data)` to avoid processing the last line twice.
    

### 3. Math & Limits

- **`std::round`**: Standard rounding (0.5 goes away from zero).
    
- **`std::numeric_limits<T>::max()`**: The standard way to get the largest possible value for a type.
    
- **`std::clamp`**: Keeps a value within a range (e.g., $[0, 255]$ for colors).
    

---

## 🎨 Graphics Context (Renderer specific)

- **Z-buffer**: Initialized to `-std::numeric_limits<float>::max()` to represent "infinite distance" so that any new pixel is closer.
    
- **Viewport Transform**: We discussed mapping normalized coordinates ($[-1, 1]$) to screen coordinates ($[0, \text{width}]$) using floating point math.
    
- **Color Initialization**: Using `{R, G, B, A}` aggregate initialization for `TGAColor` to set clear colors (like your light blue-grey background).