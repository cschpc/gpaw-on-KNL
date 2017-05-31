# GPAW on KNL

## GPAW

[GPAW](https://wiki.fysik.dtu.dk/gpaw/) is a density-functional theory (DFT)
program for ab initio electronic structure calculations using the projector
augmented wave method. It is written mostly in Python and uses MPI for
parallelisation.

GPAW is licensed under GPL and is freely available at:
  https://gitlab.com/gpaw/gpaw

## Porting

### Tuning of memory allocation

Standard memory allocators (such as ```malloc```) lock the memory pool when
doing an allocation to avoid race conditions. In contrast, Intel TBB scalable
allocator (```tbbmalloc```) uses thread-local memory pools that avoid
repeated locking of the global memory pool. In essence, multiple memory
allocations are replaced with a single memory allocation and some internal
book keeping.

Linux kernel supports very large memory pages in the form of huge pages. Huge
pages are a pool of pre-allocated, non-swappable memory that can be used to
allocate memory in very large chunks (e.g. 2M) instead of the typical default
of 4k chunks. In general, increased memory consumption is traded for a
potential gain in performance. In combination with tbbmalloc, huge pages
offer an attractive (and easily tested) option to possibly streamline memory
allocation in parallel programs.

Even though GPAW, as a Python program, is not multi-threaded, it seems that
on KNLs it is clearly beneficial to GPAW's performance to combine tbbmalloc
with huge pages. **Up to 5% faster run times are observed for GPAW when both
tbbmalloc and huge pages are in use.** The actual size of the huge pages does
not seem to be significant (tested with 2M, 4M, 8M, and 16M page sizes).

In order to use tbbmalloc and huge pages, one can e.g. set the following
environment variables:
```
export LD_PRELOAD=$TBBROOT/lib/intel64/gcc4.7/libtbbmalloc_proxy.so.2
export LD_PRELOAD=$LD_PRELOAD:$TBBROOT/lib/intel64/gcc4.7/libtbbmalloc.so.2
export TBB_MALLOC_USE_HUGE_PAGES=1
```


## Performance

### Test cases

Two slightly different benchmarks were used as test cases to study the
performance of GPAW on KNLs. One of the benchmarks (Carbon nanotube) is aimed
at smaller systems (up to 10 nodes) while the other one (Copper filament) is
aimed at larger systems (up to 100 nodes). Both test cases were used to study
the performance and scaling properties of GPAW up to 8 KNL nodes.

The benchmarks are available at:
  https://github.com/mlouhivu/gpaw-benchmarks.git

Default input parameters were used for both benchmarks.

#### Case 1: Carbon nanotube

A ground state calculation for a carbon nanotube in vacuum. By default uses a
6-6-10 nanotube with 240 atoms (freely adjustable) and serial LAPACK with an
option to use ScaLAPACK.

Input file: carbon-nanotube/input.py

#### Case 2: Copper filament

A ground state calculation for a copper filament in vacuum. By default uses a
2x2x3 FCC lattice with 71 atoms (freely adjustable) and ScaLAPACK for
parallelisation.

Input file: copper-filament/input.py

### Run parameters

No special command line options or environment variables are needed to run the
benchmarks on KNLs. One can simply say e.g.
```
mpirun -np 256 gpaw-python input.py
```

If not linked at build time, one may also enable the optimised memory
allocator (```tbbmalloc```) of Intel TBB by setting environment variable
```LD_PRELOAD``` to point to the correct libraries, i.e. for example:
```
export LD_PRELOAD=$TBBROOT/lib/intel64/gcc4.7/libtbbmalloc_proxy.so.2
export LD_PRELOAD=$LD_PRELOAD:$TBBROOT/lib/intel64/gcc4.7/libtbbmalloc.so.2
```

It may also be beneficial to use hugepages together with ```tbbmalloc```:
```
export TBB_MALLOC_USE_HUGE_PAGES=1
```

### Results

### Performance comparison

```
#
# GPAW runtimes for the SCF-cycle (in seconds).
#

# Case 1: Carbon nanotube
#   A ground state calculation for a 6-6-10 carbon nanotube in vacuum
#   using serial LAPACK.
#
# Case 2: Copper filament
#   A ground state calculation for a copper filament (2x2x3 FCC lattice)
#   in vacuum using ScaLAPACK.
#

# Columns:
#   n   -- number of CPUs / GPUs / MICs
#   cpu -- runtime (in seconds) for CPUs
#          Intel Xeon Haswell (E5-2690v3)
#   knc -- runtime (in seconds) for KNCs
#          Intel Xeon Phi 7120P
#          (host) Intel Xeon Haswell (E5-2680v3)
#   knl -- runtime (in seconds) for KNLs
#          Intel Xeon Phi 7210
#   k40 -- runtime (in seconds) for K40 GPUs
#          NVIDIA Tesla K40
#          (host) Intel Xeon Ivy Bridge (E5-2620-v2)
#   k80 -- runtime (in seconds) for K40 GPUs
#          NVIDIA Tesla K80
#          (host) Intel Xeon Haswell (E5-2680v3)

#[ Carbon nanotube (6,6,10) ]
#: n cpu knc knl k40 k80
1 517.802 nan 319.925 237.560 249.446
2 253.896 281.230 206.609 nan nan
4 136.237 164.901 141.332 nan nan
8 80.676 102.869 101.327 nan nan
12 nan 87.047 nan nan nan
16 55.597 77.004 nan nan nan
24 nan 68.275 nan nan nan
32 41.652 63.805 nan nan nan
64 30.635 nan nan nan nan

#[ Copper filament (223,8) ]
# Note: the GPU times (k40, k80) are scaled up due to slightly different
#       run parameters (grid size 0.28 vs. the default 0.22)
#: n cpu knc knl k40 k80
1 802.238 nan 323.388 386.770 344.371
2 405.808 nan 172.313 nan nan
4 195.167 nan 126.997 nan nan
8 95.362 102.607 79.952 nan nan
12 nan nan nan nan nan
16 60.294 69.441 nan nan nan
24 nan 53.774625 nan nan nan
32 33.693 42.829 nan nan nan
64 19.287 31.695 nan nan nan


#
# Case 2 (copper filament) didn't have enough memory on GPUs using default
# parameters. Grid size was increased from 0.22 to 0.28 to alleviate this.
# For the comparison, the results need to be scaled up to account for this
# using the CPU result as a yardstick (scaling factor q=2.113216). Below
# are the raw numbers for both CPUs and GPUs.
#
#[ Copper filament (223,8 / grid 0.28) ]
#: n cpu knc knl k40 k80
1 379.629 nan nan 183.025 162.961
```

## Conclusions

