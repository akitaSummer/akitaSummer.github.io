---
title: 使用Rust+WebAssembly来辅助你的JavaScript
date: 2022-06-08 00:41:29
tags:
  - WebAssembly
  - rust
categories: 学习笔记
---

> Measure first. Optimize. Measure again.

# 前言

之前编写的 rust 在前端的场景应用不多，我在最近研究了一下 WebAssembly 的实际落地，最终给一个开源计算库使用 WebAssembly 重写了计算逻辑，加速了 7x，开发时遇到了很多细节优化点，在这里分享给大家

# 初版构建

这个开源项目的迭代部分如下

```javascript
var matrix;

// dosomething...

for (i = 0; i < maxIterations && !converged; i++) {
  converged = iterate(settings, matrix).converged;
}

// dosomething...

return matrix;
```

其中的 iterate 函数为核心计算功能，具体内容如下。

```javascript
// iterate.js
/**
 * Graphology Noverlap Iteration
 * ==============================
 *
 * Function used to perform a single iteration of the algorithm.
 */

/**
 * Matrices properties accessors.
 */
var NODE_X = 0,
  NODE_Y = 1,
  NODE_SIZE = 2;

/**
 * Constants.
 */
var PPN = 3;

/**
 * Helpers.
 */
function hashPair(a, b) {
  return a + "§" + b;
}

function jitter() {
  return 0.01 * (0.5 - Math.random());
}

/**
 * Function used to perform a single interation of the algorithm.
 *
 * @param  {object}       options    - Layout options.
 * @param  {Float32Array} NodeMatrix - Node data.
 * @return {object}                  - Some metadata.
 */
module.exports = function iterate(options, NodeMatrix) {
  // Caching options
  var margin = options.margin;
  var ratio = options.ratio;
  var expansion = options.expansion;
  var gridSize = options.gridSize; // TODO: decrease grid size when few nodes?
  var speed = options.speed;

  // Generic iteration variables
  var i, j, x, y, l, size;
  var converged = true;

  var length = NodeMatrix.length;
  var order = (length / PPN) | 0;

  var deltaX = new Float32Array(order);
  var deltaY = new Float32Array(order);

  // Finding the extents of our space
  var xMin = Infinity;
  var yMin = Infinity;
  var xMax = -Infinity;
  var yMax = -Infinity;

  for (i = 0; i < length; i += PPN) {
    x = NodeMatrix[i + NODE_X];
    y = NodeMatrix[i + NODE_Y];
    size = NodeMatrix[i + NODE_SIZE] * ratio + margin;

    xMin = Math.min(xMin, x - size);
    xMax = Math.max(xMax, x + size);
    yMin = Math.min(yMin, y - size);
    yMax = Math.max(yMax, y + size);
  }

  var width = xMax - xMin;
  var height = yMax - yMin;
  var xCenter = (xMin + xMax) / 2;
  var yCenter = (yMin + yMax) / 2;

  xMin = xCenter - (expansion * width) / 2;
  xMax = xCenter + (expansion * width) / 2;
  yMin = yCenter - (expansion * height) / 2;
  yMax = yCenter + (expansion * height) / 2;

  // Building grid
  var grid = new Array(gridSize * gridSize),
    gridLength = grid.length,
    c;

  for (c = 0; c < gridLength; c++) grid[c] = [];

  var nxMin, nxMax, nyMin, nyMax;
  var xMinBox, xMaxBox, yMinBox, yMaxBox;

  var col, row;

  for (i = 0; i < length; i += PPN) {
    x = NodeMatrix[i + NODE_X];
    y = NodeMatrix[i + NODE_Y];
    size = NodeMatrix[i + NODE_SIZE] * ratio + margin;

    nxMin = x - size;
    nxMax = x + size;
    nyMin = y - size;
    nyMax = y + size;

    xMinBox = Math.floor((gridSize * (nxMin - xMin)) / (xMax - xMin));
    xMaxBox = Math.floor((gridSize * (nxMax - xMin)) / (xMax - xMin));
    yMinBox = Math.floor((gridSize * (nyMin - yMin)) / (yMax - yMin));
    yMaxBox = Math.floor((gridSize * (nyMax - yMin)) / (yMax - yMin));

    for (col = xMinBox; col <= xMaxBox; col++) {
      for (row = yMinBox; row <= yMaxBox; row++) {
        grid[col * gridSize + row].push(i);
      }
    }
  }

  // Computing collisions
  var cell;

  var collisions = new Set();

  var n1, n2, x1, x2, y1, y2, s1, s2, h;

  var xDist, yDist, dist, collision;

  for (c = 0; c < gridLength; c++) {
    cell = grid[c];

    for (i = 0, l = cell.length; i < l; i++) {
      n1 = cell[i];

      x1 = NodeMatrix[n1 + NODE_X];
      y1 = NodeMatrix[n1 + NODE_Y];
      s1 = NodeMatrix[n1 + NODE_SIZE];

      for (j = i + 1; j < l; j++) {
        n2 = cell[j];
        h = hashPair(n1, n2);

        if (gridLength > 1 && collisions.has(h)) continue;

        if (gridLength > 1) collisions.add(h);

        x2 = NodeMatrix[n2 + NODE_X];
        y2 = NodeMatrix[n2 + NODE_Y];
        s2 = NodeMatrix[n2 + NODE_SIZE];

        xDist = x2 - x1;
        yDist = y2 - y1;
        dist = Math.sqrt(xDist * xDist + yDist * yDist);
        collision = dist < s1 * ratio + margin + (s2 * ratio + margin);

        if (collision) {
          converged = false;

          n2 = (n2 / PPN) | 0;

          if (dist > 0) {
            deltaX[n2] += (xDist / dist) * (1 + s1);
            deltaY[n2] += (yDist / dist) * (1 + s1);
          } else {
            // Nodes are on the exact same spot, we need to jitter a bit
            deltaX[n2] += width * jitter();
            deltaY[n2] += height * jitter();
          }
        }
      }
    }
  }

  for (i = 0, j = 0; i < length; i += PPN, j++) {
    NodeMatrix[i + NODE_X] += deltaX[j] * 0.1 * speed;
    NodeMatrix[i + NODE_Y] += deltaY[j] * 0.1 * speed;
  }

  return { converged: converged };
};
```

然后我使用了 wasm-pack 构建了 rust 项目，并写代码如下。

```rust
mod utils;

use std::collections::HashSet;

use rand::prelude::*;
use wasm_bindgen::prelude::*;

use js_sys::Float32Array;

// When the `wee_alloc` feature is enabled, use `wee_alloc` as the global
// allocator.
#[cfg(feature = "wee_alloc")]
#[global_allocator]
static ALLOC: wee_alloc::WeeAlloc = wee_alloc::WeeAlloc::INIT;

const NODE_X: usize = 0;
const NODE_Y: usize = 1;
const NODE_SIZE: usize = 2;

const PPN: u32 = 3;

fn hashPair(a: usize, b: usize) -> String {
    return format!("{:?}§{:?}", a, b);
}

fn jitter() -> f32 {
    let mut rng = rand::thread_rng();
    let y: f32 = rng.gen();
    return 0.01 * (0.5 - y);
}

#[wasm_bindgen]
pub struct Res {
    matrix: Vec<f32>,
    converged: bool,
}

#[wasm_bindgen]
impl Res {
    #[wasm_bindgen(constructor)]
    pub fn new(matrix: Vec<f32>, converged: bool) -> Self {
        Res { matrix, converged }
    }

    pub fn get_node_matrix(&self) -> Float32Array {
        unsafe { Float32Array::view(&self.matrix) }
    }
}

#[wasm_bindgen]
pub fn iterate(
    margin: f32,
    ratio: f32,
    expansion: f32,
    grid_size: f32,
    speed: f32,
    mut node_matrix: Vec<f32>,
) -> Res {
    let length: u32 = node_matrix.len() as u32;
    let order = ((length / PPN) | 0) as usize;

    let mut delta_x = vec![0.0 as f32; order];
    let mut delta_y = vec![0.0 as f32; order];

    let mut x_min = f32::INFINITY;
    let mut y_min = f32::INFINITY;
    let mut x_max = f32::NEG_INFINITY;
    let mut y_max = f32::NEG_INFINITY;

    {
        let mut i = 0;

        while i < length {
            let index = i as usize;
            let x = unsafe { *node_matrix.get_unchecked(index + NODE_X) };
            let y = unsafe { *node_matrix.get_unchecked(index + NODE_Y) };
            let size = unsafe { *node_matrix.get_unchecked(index + NODE_SIZE) * ratio + margin };

            x_min = f32::min(x_min, x - size);

            y_min = f32::min(y_min, y - size);

            x_max = f32::max(x_max, x + size);

            y_max = f32::max(y_max, x + size);

            i += PPN;
        }
    }

    let width = x_max - x_min;
    let height = y_max - y_min;
    let x_center = (x_max + x_min) / 2.0;
    let y_center = (y_max + y_min) / 2.0;

    x_min = (x_center as f64 - (expansion as f64 * width as f64) / 2.0) as f32;
    x_max = (x_center as f64 + (expansion as f64 * width as f64) / 2.0) as f32;
    y_min = (y_center as f64 - (expansion as f64 * height as f64) / 2.0) as f32;
    y_max = (y_center as f64 + (expansion as f64 * height as f64) / 2.0) as f32;

    let grid_length = (grid_size * grid_size) as usize;

    let mut grid = vec![Vec::<u32>::new(); grid_length];

    {
        let mut i = 0;
        while i < length {
            let index = i as usize;
            let x = unsafe { *node_matrix.get_unchecked(index + NODE_X) };
            let y = unsafe { *node_matrix.get_unchecked(index + NODE_Y) };
            let size = unsafe { *node_matrix.get_unchecked(index + NODE_SIZE) * ratio + margin };

            let nx_min = x - size;
            let nx_max = x + size;
            let ny_min = y - size;
            let ny_max = y + size;

            let x_min_box = f32::floor((grid_size * (nx_min - x_min)) / (x_max - x_min));
            let x_max_box = f32::floor((grid_size * (nx_max - x_min)) / (x_max - x_min));
            let y_min_box = f32::floor((grid_size * (ny_min - y_min)) / (y_max - y_min));
            let y_max_box = f32::floor((grid_size * (ny_max - y_min)) / (y_max - y_min));

            {
                let mut col = x_min_box;
                while col <= x_max_box {
                    let mut row = y_min_box;
                    while row <= y_max_box {
                        grid[(col * grid_size + row) as usize].push(i);
                        row += 1.0;
                    }
                    col += 1.0;
                }
            }

            i += PPN;
        }
    }

    let mut converged = true;

    {
        let mut collisions = HashSet::<String>::new();
        let mut c = 0;
        while c < grid_length {
            let cell = unsafe { grid.get_unchecked(c) };
            let cell_length = cell.len() as usize;

            let mut i = 0;

            while i < cell_length {
                let n1 = unsafe { *cell.get_unchecked(i) as usize };

                let x1 = unsafe { *node_matrix.get_unchecked(n1 + NODE_X) };
                let y1 = unsafe { *node_matrix.get_unchecked(n1 + NODE_Y) };
                let s1 = unsafe { *node_matrix.get_unchecked(n1 + NODE_SIZE) };

                let mut j = i + 1;
                while j < cell_length {
                    let n2 = unsafe { *cell.get_unchecked(j) as usize };
                    let h = hashPair(n1, n2);
                    if grid_length > 1 && collisions.contains(&h) {
                        j += 1;
                        continue;
                    }

                    if grid_length > 1 {
                        collisions.insert(h);
                    }

                    let x2 = unsafe { *node_matrix.get_unchecked(n2 + NODE_X) };
                    let y2 = unsafe { *node_matrix.get_unchecked(n2 + NODE_Y) };
                    let s2 = unsafe { *node_matrix.get_unchecked(n2 + NODE_SIZE) };

                    let x_dist = x2 - x1;
                    let y_dist = y2 - y1;
                    let dist = f32::sqrt(x_dist * x_dist + y_dist * y_dist);
                    let collision = dist < s1 * ratio + margin + (s2 * ratio + margin);
                    if collision {
                        converged = false;

                        let n2 = ((n2 as u32 / PPN) | 0) as usize;
                        if dist > 0.0 {
                            unsafe {
                                *delta_x.get_unchecked_mut(n2) += (x_dist / dist) * (1.0 + s1);
                                *delta_y.get_unchecked_mut(n2) += (y_dist / dist) * (1.0 + s1);
                            };
                        } else {
                            unsafe {
                                *delta_x.get_unchecked_mut(n2) += width * jitter();
                                *delta_y.get_unchecked_mut(n2) += height * jitter();
                            };
                        }
                    }

                    j += 1;
                }

                i += 1;
            }

            c += 1;
        }
    }

    {
        let mut i = 0;
        let mut j = 0;
        while i < length {
            let index = i as usize;
            unsafe {
                *node_matrix.get_unchecked_mut(index + NODE_X) +=
                    *delta_x.get_unchecked(j) * 0.1 * speed;
                *node_matrix.get_unchecked_mut(index + NODE_Y) +=
                    *delta_y.get_unchecked(j) * 0.1 * speed;
            }
            i += PPN;
            j += 1;
        }
    }

    Res {
        matrix: node_matrix,
        converged,
    }
}
```

将 js 文件更改为

```javascript
var matrix;

// dosomething...

if (settings.wasm) {
  for (i = 0; i < maxIterations && !converged; i++) {
    var res = test_iterate(
      settings.margin,
      settings.ratio,
      settings.expansion,
      settings.gridSize,
      settings.speed,
      matrix
    );
    converged = res.converged;
    matrix = res.get_node_matrix();
  }
} else {
  for (i = 0; i < maxIterations && !converged; i++) {
    converged = iterate(settings, matrix).converged;
  }
}

// dosomething...

return matrix;
```

然后我跑了一下 benchmark，发现 js 版本用时 700ms，wasm 版本用时 1.4s，wasm 比 js 跑的慢太多了，于是我开始了漫漫优化之旅。

# 不要跟 js 比拼字符串操作

逐行断点后，我发现，wasm 的主要性能消耗在了 hashPair 函数上，而他的目的是为了制作 set 的 hash 值，在 js 中，并没有元组这个概念，所以我们需要拼字符串作为 hash，而 rust 中有元组，我们可以使用元组作为 hash 值存入 set 中。

```rust
fn hashPair(a: usize, b: usize) -> String {
    return format!("{:?}§{:?}", a, b);
}


let mut collisions = HashSet::<String>::new();

let h = hashPair(n1, n2);
if grid_length > 1 && collisions.contains(&h) {
    j += 1;
    continue;
 }

// 删除hashPair，并更改为元组
let mut collisions = HashSet::<(usize, usize)>::new();

let h = (n1, n2);
if grid_length > 1 && collisions.contains(&h) {
    j += 1;
    continue;
 }
```

效果显著，直接从 1.4s 降低至 240ms。

# 使用指针减少通信间的数据转换

众所周知，js 的数据结构和 rust 的不相同，而我们的初版中存在大量的 js 和 rust 的数据交互，这样会频繁的将 js 的`Float32Array`与 rust 的`Vec<f32>`转换，这里会耗费一些性能：

```javascript
var matrix;

// dosomething...

if (settings.wasm) {
  for (i = 0; i < maxIterations && !converged; i++) {
    var res = test_iterate(
      settings.margin,
      settings.ratio,
      settings.expansion,
      settings.gridSize,
      settings.speed,
      matrix
    );
    converged = res.converged;
    matrix = res.get_node_matrix();
  }
} else {
  for (i = 0; i < maxIterations && !converged; i++) {
    converged = iterate(settings, matrix).converged;
  }
}

// dosomething...

return matrix;
```

我们可以使用传递指针地址的方式来减少`Float32Array`与 rust 的`Vec<f32>`转换带来的性能损失。

```rust
#![feature(box_syntax)]

// 储存matrix和converged的对象
pub struct Context {
    pub matrix: Vec<f32>,
    pub converged: bool,
}

impl Context {
    pub fn new(matrix: Vec<f32>) -> Self {
        Context {
            matrix,
            converged: true,
        }
    }
}

// 获取Context的指针地址
#[wasm_bindgen]
pub fn create_computer_ctx(x: Vec<f32>) -> usize {
    let ctx = Context::new(x);
    Box::into_raw(box ctx) as _
}

// 根据Context的指针地址获取Float32Array
#[wasm_bindgen]
pub fn get_node_matrix(ctx: *mut Context) -> Float32Array {
    let ctx = unsafe { &mut *ctx };
    unsafe { Float32Array::view(&ctx.matrix) }
}

// 根据Context的指针地址获取converged
#[wasm_bindgen]
pub fn get_converged(ctx: *mut Context) -> bool {
    let ctx = unsafe { &mut *ctx };
    ctx.converged
}


#[wasm_bindgen]
pub fn iterate(
    margin: f32,
    ratio: f32,
    expansion: f32,
    grid_size: f32,
    speed: f32,
    ctx: *mut Context, // Context指针地址
) {
    let ctx = unsafe { &mut *ctx }; // 根据指针地址获取Context对象
    let mut node_matrix = &mut ctx.matrix;

    // ...

 }
```

js 侧修改为

```javascript
if (settings.wasm) {
  // 获取指针值
  var ctxPtr = create_computer_ctx(matrix);
  for (i = 0; i < maxIterations && !converged; i++) {
    test_iterate(
      settings.margin,
      settings.ratio,
      settings.expansion,
      settings.gridSize,
      settings.speed,
      ctxPtr
    );
    converged = get_converged(ctxPtr);
  }
  // 根据指针地址获取matrix
  matrix = get_node_matrix(ctxPtr);
} else {
  for (i = 0; i < maxIterations && !converged; i++) {
    converged = iterate(settings, matrix).converged;
  }
}
```

需要注意的是，想要使用`#![feature(box_syntax)]`，需要在`cargo.toml`中，将 wasm-bindgen 支持 nightly

```toml
wasm-bindgen = { version = "=0.2.73", features = ["nightly"] }
```

wasm-pack 并不支持 nightly，所以你需要去运行`rustup run nightly $HOME/.cargo/bin/wasm-pack "build" --release --target nodejs`才能构建成功
测试后有了 10%的性能提升。

# 过于复杂的宏并不好

我在创建一个二维数组时，采用的是宏来编写

```rust
let mut grid = vec![Vec::<u32>::new(); grid_length];
```

但是当我手动编写后，性能提升了 4%

```rust
let mut grid = Vec::<Vec<u32>>::new();

for n in 0..grid_length {
    grid.push(Vec::<u32>::new());
}
```

# 用 SIMD 来榨干你的 CPU

SIMD（Single Instruction Multiple Data）中文为单指令多数据，它是一种特的 CPU 指令，能够实现数据的并行处理，在音视频和图像处理领域有广泛的使用。与之相反的概念是 SISD （Single Instruction Single Data），每次数据计算都需要执行一次指令。
简单来说，之前需要四条指令才能完成的计算，使用 SIMD 计算只需要一条指令即可完成：

![simd](/image/rust-wasm/simd.png)
遗憾的是 js 并不支持 SIMD，但是在 wasm 中我们却可以使用。
我们可以在我们的循环计算中使用 SIMD 来优化

```rust
#![feature(box_syntax, wasm_simd)]
#[cfg(feature = "simd")]
use std::arch::wasm32::*;

// ...

{
        let mut i = 0;
        while i < length {
            let index = i as usize;
            let x = unsafe { *node_matrix.get_unchecked(index + NODE_X) };
            let y = unsafe { *node_matrix.get_unchecked(index + NODE_Y) };
            let size = unsafe { *node_matrix.get_unchecked(index + NODE_SIZE) * ratio + margin };

            let nx_min = x - size;
            let nx_max = x + size;
            let ny_min = y - size;
            let ny_max = y + size;

            // let x_min_box = f32::floor((grid_size * (nx_min - x_min)) / (x_max - x_min));
            // let x_max_box = f32::floor((grid_size * (nx_max - x_min)) / (x_max - x_min));
            // let y_min_box = f32::floor((grid_size * (ny_min - y_min)) / (y_max - y_min));
            // let y_max_box = f32::floor((grid_size * (ny_max - y_min)) / (y_max - y_min));

            let n = f32x4(nx_min, nx_max, ny_min, ny_max);
            let size = f32x4(grid_size, grid_size, grid_size, grid_size);
            let f = f32x4(nx_min, nx_max, ny_min, ny_max);
            let min = f32x4(x_min, x_min, y_min, y_min);
            let max = f32x4(x_max, x_max, y_max, y_max);

            let n_s_i = f32x4_sub(n, min);
            let a_s_i = f32x4_sub(max, min);
            let s_m_n_s_i = f32x4_mul(size, n_s_i);
            let d = f32x4_div(s_m_n_s_i, a_s_i);

            let (x_min_box, x_max_box, y_min_box, y_max_box) = unsafe {
                (
                    f32x4_extract_lane::<0>(d),
                    f32x4_extract_lane::<1>(d),
                    f32x4_extract_lane::<2>(d),
                    f32x4_extract_lane::<3>(d),
                )
            };

            let x_min_box = f32::floor(x_min_box);
            let x_max_box = f32::floor(x_max_box);
            let y_min_box = f32::floor(y_min_box);
            let y_max_box = f32::floor(y_max_box);

            {
                let mut col = x_min_box;
                while col <= x_max_box {
                    let mut row = y_min_box;
                    while row <= y_max_box {
                        grid[(col * grid_size + row) as usize].push(i);
                        row += 1.0;
                    }
                    col += 1.0;
                }
            }

            i += PPN;
        }
    }
```

当然，这个项目的这种计算较少，速度也仅仅只提升了 5%左右

# 更快的数据结构

set 的查询，导致了 wasm 的速度降低，在 rust 我们可以选用更好的 set 结构，来加速我们的 set 查询

```rust
use tinyset::Set;

// let mut collisions = HashSet::<(usize, usize)>::new();
let mut collisions = Set::<(usize, usize)>::new();
```

替换后，我们的 wasm 又快了一倍

# 更小的 WASM

其实当你使用 wasm-pack 起了一个 wasm 的项目后，他会自动帮助你引入一个 wee_alloc 的包，而一般他会位于你的项目顶部

```rust
#[cfg(feature = "wee_alloc")]
#[global_allocator]
static ALLOC: wee_alloc::WeeAlloc = wee_alloc::WeeAlloc::INIT;
```

这样，当你运行`wasm-pack build --features wee_alloc`时会打包一个更小的 wasm。
不过我们的项目因为使用了 nightly，所以我们的运行方式是`rustup run nightly $HOME/.cargo/bin/wasm-pack "build" --release --target nodejs -- --features wee_alloc`

# 有效果但没在项目中用到的

- 昂贵的 f32::is_finite()
  - 这个函数的底层实现逻辑是

```rust
self.abs_private() < Self::INFINITY

f32::from_bits(self.to_bits() & 0x7fff_ffff)
```

如果你能用与 Linux 的 perf 分析工具检测的话就会发现，f32::is_finite()的消耗时间较多，具体原因不详，可能与浮点数跟 xmm 寄存器的处理有关系，如果你能保证你的 f32 是>0 的话，请使用`== std::f32::INFINITY`来进行处理

- 不太行的 f32x4.max
  - f32x4.max 会校验 NaN 的问题，这会导致计算速度变慢，你可以使用 f32x4.lt/f32x4.or 或者未来将支持的 f32x4.pmax 来规避这个问题

# 没了

真没了，就这点东西，贼简单。
另外，其实在这种 js 与 WebAssembly 通信较少的场景下，使用 WebAssembly 进行加速，效果还是比较显著的，如果通信比较频繁，我个人推荐使用第三章的指针地址传递的方式来优化，不过效果如何，还待检测。
