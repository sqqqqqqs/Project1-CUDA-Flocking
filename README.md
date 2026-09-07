**University of Pennsylvania, CIS 5650: GPU Programming and Architecture,
Project 1 - Flocking**

* Yikai Li
* Tested on: Razer Blade 16 (RZ09-0528), Windows 11 Home 25H2 64-bit, AMD Ryzen AI 9 365 with Radeon 880M (10 cores), 32 GB LPDDR5-8000 RAM, NVIDIA GeForce RTX 5080 Laptop GPU 16 GB (Personal Computer)

![CUDA Boids simulation](images/boids.png)
![CUDA Boids animation](images/boids.gif)
## Overview

This project implements Reynolds-style boid flocking with CUDA using cohesion, separation, and alignment. It includes three neighbor-search methods:

- **Naive:** every boid checks every other boid.
- **Scattered grid:** boids are grouped by grid cell using sorted indices, but their position and velocity data remain scattered.
- **Coherent grid:** position and velocity data are reordered by grid cell for more contiguous memory access.

Separate input and output velocity buffers ensure that every boid reads a consistent state during each simulation step.

## Performance Analysis

### Methodology

- Build configuration: **Release x64**
- Vertical synchronization: **Off**
- NVIDIA Max Frame Rate: **Off**
- Time step: **0.2**
- Default block size: **128 threads**
- Default grid configuration: **8-cell search with cell width $2r$**
- The scaling graphs use one stabilized FPS reading per configuration after warm-up.
- All comparisons used the same machine and executable window size.

### Effect of boid count without visualization

![Performance without visualization](images/performance_no_visualization.png)

| Boids | Naive FPS | Scattered FPS | Coherent FPS |
|---:|---:|---:|---:|
| 5,000 | 851.0 | 1144.0 | 1224.0 |
| 10,000 | 519.0 | 1124.0 | 1205.0 |
| 25,000 | 251.0 | 1063.0 | 1173.0 |
| 50,000 | 102.0 | 860.0 | 1167.0 |
| 100,000 | 25.2 | 589.0 | 1069.0 |
| 200,000 | 8.2 | 281.0 | 634.0 |
| 500,000 | 1.4 | 77.3 | 445.0 |
| 1,000,000 | — | 20.6 | 187.0 |

Naive performance decreases rapidly because doubling the boid count approximately quadruples the number of pairwise checks. The uniform-grid methods scale much better by limiting the search to nearby cells. They eventually slow as well because a fixed simulation volume causes more boids to occupy each cell. At one million boids, the coherent grid was about **9.1 times faster** than the scattered grid because its neighbor data is stored contiguously.

### Effect of visualization

![Performance with visualization](images/performance_with_visualization.png)

| Boids | Naive FPS | Scattered FPS | Coherent FPS |
|---:|---:|---:|---:|
| 5,000 | 672.0 | 873.0 | 959.0 |
| 10,000 | 414.0 | 824.0 | 955.0 |
| 25,000 | 216.0 | 799.0 | 917.0 |
| 50,000 | 95.4 | 675.0 | 924.0 |
| 100,000 | 23.9 | 512.0 | 879.0 |
| 200,000 | 7.6 | 241.0 | 554.0 |
| 500,000 | 1.2 | 73.3 | 383.0 |
| 1,000,000 | — | 20.2 | 151.8 |

Visualization lowers FPS because each frame also updates the graphics buffer and renders the particles. The same overall trend remains: Naive search falls quickly, while both grid methods support much larger simulations. At high boid counts the coherent grid retains the largest advantage.

### Effect of block size and block count

This experiment used **50,000 boids** with visualization disabled. The number of blocks is $\lceil N / \text{blockSize} \rceil$. The graph normalizes each implementation to its own fastest result so the effect of block size is visible despite the algorithms' different absolute framerates.

![Block size performance](images/block_size_performance.png)

| Threads per block | Block count | Naive FPS | Scattered FPS | Coherent FPS |
|---:|---:|---:|---:|---:|
| 32 | 1563 | 68.9 | 795 | 1117 |
| 128 | 391 | 96.9 | 830 | 1082 |
| 512 | 98 | 95.9 | 845 | 1158 |
| 1024 | 49 | 97.6 | 829 | 1135 |

The Naive implementation was substantially slower at 32 threads per block. The grid implementations changed by less than 7% across the tested sizes and both reached their highest FPS at 512 threads. Increasing the block size to 1024 provided no further improvement. A block size of 128 remains a reasonable default because it performs close to the best result for all three implementations.

### Coherent versus scattered grid

In a five-run controlled test with 20,000 boids, visualization disabled, and block size 128, the coherent grid averaged **1216.2 FPS**, compared with **1076.8 FPS** for the scattered grid, an improvement of about **12.9%**.

This improvement was expected because boids in the same cell are contiguous and neighbor access no longer requires an extra particle-index lookup. The benefit is partially offset by the cost of rearranging the position and velocity arrays every frame.

### Eight-cell versus 27-cell search

A separate five-run test used 20,000 boids, visualization disabled, and block size 128.

| Grid configuration | Average FPS |
|---|---:|
| 8 cells, cell width $2r$ | 1123.8 |
| 27 cells, cell width $r$ | 1224.2 |

The 27-cell version was approximately **8.9% faster**. Although it visits more cells, each cell is smaller. The eight-cell configuration examines a candidate volume proportional to $8(2r)^3 = 64r^3$, while the 27-cell configuration examines $27r^3$. In this test, reducing candidate boids and distance calculations outweighed the overhead of checking more cell ranges.

## Build Notes

**CMake modifications:** None.