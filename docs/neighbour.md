

## Neighbour Algorithm

#### 1 Introduction 
The neighbour module is intended for use as a base module for implementations of sub-routines in the MPS code. The current serial implementation of the neighbour algorithm which is similar to the cell-linked-list approach involves a uniform cell grid onto which the particles are allocated after which the neighbour search is carried out by searching only the particles in the neighbouring cells (9 in case of 2D and 27 in case of 3D). The parallel implementation follows a [paper written by Simon Green](http://developer.download.nvidia.com/assets/cuda/files/particles.pdf) based on Particle Simulation in CUDA. The parallel algorithm is implemented in C++ CUDA and uses the CUDA Thrust library for the *RadixSort* operation. There are two different parallel neighbour search functions available depending upon the global memory consumption and memory transfer capabilities of the connector bus. A serial imeplementation is also available. 

#### 2 How to use the module?

The serial neighbour module can be imported via headers like so - `#include<neighb.h>` 
The parallel neighbour module can be imported via headers like so - `#include<neighb.cu>`

##### 2.1 The first function available is `neighbour_cuda_1(args)` 
The required arguments are :
- int* - `x`, `y`, `z`
- int - `Xmax`, `Xmin`, `Ymax`, `Ymin`, `Zmax`, `Zmin`, `re`, `DELTA`, `NUM`, `MAX_NEIGHB`
- int** - `neighb`

The function modifies the `neighb` array which is passed by reference in the arguments and populates it with the required values of particle IDs and number of neighbours. For any particle ID `i` the number of neighbours can be found by querying `neighb[i][1]`. And all the neighbour particle IDs can be iterated over by doing the following. 

```C++
for(int j=0; j<neighb[i][1]; j++){
  neighb[i][j+2];
}
```

##### 2.2 There is another function available which is called `neighbour_cuda_2(args)`
The required arguments are :
- int* - `x`, `y`, `z`, `particleHash`, `particleID`, `cellStart`, `cellEnd`
- int - `Xmax`, `Xmin`, `Ymax`, `Ymin`, `Zmax`, `Zmin`, `re`, `DELTA`, `NUM`, `MAX_NEIGHB`

This function modifies `particleHash`, `particleID`, `cellStart`, `cellEnd` just like the previous function modifies `neighb` but these 4 arrays combined take less space on the global memory than the `neighb` array and hence constitute a preferable mode of implementation. Together, they make up O(2N + 2M) space where N is the total number of particles and M is the total number of cells. Whereas the `neighb` takes up O(N * MAX_NEIGHB) space.

The neighbours in this can can be looped over for all particles like so:
- For an index `i` find the Particle Id and particle cell number from `particleiD` and `particleHash`. 
- Find the coordinates of the cell in terms of `i`, `j`, and `k`. Here we use `Cnum = (i-1) + (j-1)*ncx + (k-1)*ncx*ncy`. 
- Find the neighbouring cell numbers and iterate over the particles in those cells using `cellStart`, `cellEnd`, and `particleId`. `cellstart` and `cellEnd` are already populated according to the key-sorted `particleHash` with `particleId` as the key-array.

#### 3 Performance Measures of the code 

**A time study for NNS using CUDA and Serial Code**

The time study is done using a separate code which generates random particles in a domain and then Cuda NNS and serial NNS is operated on that domain. The resulting `neighb` arrays are compared for *number of neighbours* and *particle ids*. As of now, all the tests are passing. The time study is done using the `chrono` module in C++. The functions used are `neighbour_cuda_1()` and `NEIGHBOUR_serial()`. The time is averaged over three trials. 

Number of threads per block are currently set to **512** using `THREADS_PER_BLOCK` definition. Also, the maximum number of neighbours is restricted to **1500** using `MAX_NEIGHB` variable. Both of these can be changed if required.

The first plot depicts the time taken for the neighbour serach and data allocation to the `neighb` variable by both the functions. The second plot depicts the variation of factor by which the CUDA code is faster than the serial code. As can be seen the CUDA code can get more than **200x** speedup with large number of particles (**~50000**) for this particular NNS algorithm. 

![alt text](https://github.com/deeptavker/MPS/blob/master/analysis/pics/time.png)

![alt text](https://github.com/deeptavker/MPS/blob/master/analysis/pics/speedup.png)

For a case of 3D landslide, the overall speedup is over **1.45x** which is not much less than the theoretical speedup of **1.66x** if the neighbour search is considered to consume **40%** of the computation time and the GPU essentially blazes through the search. 