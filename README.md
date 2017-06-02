# GPAW on KNL

## GPAW

[GPAW](https://wiki.fysik.dtu.dk/gpaw/) is a density-functional theory (DFT)
program for ab initio electronic structure calculations using the projector
augmented wave method. It is written mostly in Python and uses MPI for
parallelisation.

GPAW is licensed under GPL and is freely available at:
  https://gitlab.com/gpaw/gpaw

## Porting

### Hardware

The code was ported to the ARCHER Knights Landing Testing and Development
Platform by Cray that consists of 12 nodes each with a single 64-core KNL
processor (Intel Xeon Phi 7210) running at 1.3GHz. Each node had 96GB of
standard memory in addition to the 16GB of high-bandwidth MCDRAM memory in a
KNL processor.

### Building software

Standard version of GPAW should be compiled using the Intel compile
environment with Intel MKL and Intel MPI, e.g. by following the
[generic installation instructions](https://wiki.fysik.dtu.dk/gpaw/install.html)
for GPAW. In this comparison, GPAW version 0.11.0 was used.

Example build and customisation scripts (used in the ARCHER KNL system)
are available at
[github.com/mlouhivu/build-recipes](https://github.com/mlouhivu/build-recipes/tree/master/gpaw-stack/examples/archer-knl-0.11.0)

#### Python+

Since ARCHER's KNL system has Sandy Bridge login nodes, one needs to build
GPAW and it's underlying Python stack in two steps. First, Python and the
other dependencies of GPAW should be built targeting the Sandy Bridge CPUs.

```
module swap PrgEnv-cray PrgEnv-intel/6.0.3
export CC=cc
export MPICC=cc
export CRAYPE_LINK_TYPE=dynamic
export CRAY_ADD_RPATH=yes

module swap craype-mic-knl craype-sandybridge

# ... install Python etc.
```

#### GPAW

After Python and the other dependencies are built, one can switch to target
the KNLs and proceed to build GPAW.
```
module swap craype-sandybridge craype-mic-knl
module load cray-memkind

# ... install GPAW
```

The compiler wrapper (`cc`) takes care of using the correct compiler options
for the target architecture. In the case of KNLs, it will add `-xMIC-AVX512`
to enable the AVX-512 instruction sets supported by KNLs. The module
`cray-memkind` is also needed to get support for the high-bandwidth memory.

#### TBB & huge pages

To improve performance
(see [Tuning memory allocation](#tuning-of-memory-allocation) for details),
one should also link to Intel TBB to benefit from an optimised memory
allocator (`tbbmalloc`) and use huge pages for memory allocation. This can be
done at run-time by setting the following environment variables and by loading
a corresponding module to set up the huge pages:
```
export LD_PRELOAD=$TBBROOT/lib/intel64/gcc4.7/libtbbmalloc_proxy.so.2
export LD_PRELOAD=$LD_PRELOAD:$TBBROOT/lib/intel64/gcc4.7/libtbbmalloc.so.2
export TBB_MALLOC_USE_HUGE_PAGES=1
module load craype-hugepages2M
```

## Benchmarks

### Setup

The ARCHER KNL nodes were used in cache mode (*quad_100*) with all of the
high-bandwidth MCDRAM memory used as an additional cache between the
processor and conventional memory.

The results from the 64-core KNLs (Xeon Phi 7210) were compared to results
from 12-core Haswell CPUs (Xeon E5-2690v3). A single KNL was compared to a
full node (two CPUs) to have comparable power consumptions.

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

An [example batch job script](./job.sh) for the ARCHER KNL system is also
available.

## Optimisation

### Tuning of memory allocation

Standard memory allocators (such as `malloc`) lock the memory pool when
doing an allocation to avoid race conditions. In contrast, Intel TBB scalable
allocator (`tbbmalloc`) uses thread-local memory pools that avoid
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

## Performance

GPAW runtimes were measured using two benchmarks (see [above](#test-cases) for
details) on 64-core KNLs (Xeon Phi 7210) and 12-core Haswell CPUs (Xeon
E5-2690v3). Only the runtime for the SCF cycle was used to exclude any
differences in the initialisation overheads. A single KNL was compared to a
full node (two CPUs) to have comparable power consumptions.

### Effects of TBB and huge pages

Different configurations were tested to find optimal performance on KNLs for
both benchmarks. Summary of the results for Case 1 are shown in Table 1 and
for Case 2 in Table 2. As can be seen from the results, the effect of
switching to TBB for memory allocation is either negligible (Case 1) or only
minor (Case 2) without the additional benefit of huge pages. If huge pages
are enabled, performance is increased for both benchmarks regardless of the
size of the huge pages.

**Table 1**. Average runtime in seconds for Case 1 when using *n* KNLs. Data
shown for runs using standard memory allocator (*stdlib*) as well as using
tbbmalloc without huge pages (*TBB*) and with 2M, 4M, 8M, or 16M huge pages
(*TBB + 2M* etc.).

| n | stdlib | TBB   | TBB + 2M | TBB + 4M | TBB + 8M | TBB + 16M |
| - | ------ | ----- | -------- | -------- | -------- | --------- |
| 1 | 329.2  | 330.0 | 319.9    | 320.8    | 321.1    |           |
| 2 | 215.5  | 213.2 | 206.6    |          |          |           |
| 4 |        | 145.5 | 141.3    |          |          |           |
| 8 |        |       | 101.3    | 101.2    | 101.3    | 101.4     |


**Table 2**. Average runtime in seconds for Case 2 when using *n* KNLs. Data
shown for runs using standard memory allocator (*stdlib*) as well as using
tbbmalloc without huge pages (*TBB*) and with 2M, 4M, 8M, or 16M huge pages
(*TBB + 2M* etc.).

| n | stdlib | TBB   | TBB + 2M | TBB + 4M | TBB + 8M |
| - | ------ | ----- | -------- | -------- | -------- |
| 1 | 341.0  | 337.1 | 323.4    | 323.9    | 323.4    |
| 2 |        | 177.7 | 172.3    | 171.0    | 170.9    |
| 4 | 133.0  | 130.7 | 127.0    | 127.0    | 127.2    |
| 8 |        |  82.2 |  80.0    |  80.0    |  80.2    |

Additionally, for Case 1 the effect of using ScaLAPACK instead of serial
LAPACK (which is the default in Case 1) was tested, but only degrading
performance was achieved.

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

