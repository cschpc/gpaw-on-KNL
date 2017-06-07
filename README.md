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

As a reference for the [performance comparisons](#performance),
CSC's Sisu supercomputer (Cray XC40) was used. In Sisu, each node has two
12-core Haswell CPUs (Intel Xeon E5-2690v3) running at 2.6GHz and 64GB of
standard DDR4 memory.

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

### Code modifications

#### Improved vectorisation with OpenMP SIMDs

The performance of GPAW on KNLs was profiled using Intel VTune Amplifier
2017 and potential targets for optimisation efforts were identified. All of
the potential targets were in the computational kernels written in C.

Two of the most prominent targets (```bmgs_fd_worker()``` in `c/bmgs/fd.c` and
```bmgs_relax()```in `c/bmgs/relax.c`) both contained triple nested loops with
single step pointer incrementations to advance the position of the input and
output arrays. To allow for better vectorisation by the compiler, OpenMP SIMD
pragmas were added and pointer incrementations was modified to be in larger
blocks at an upper loop level with explicit indeces at the lower loop level.

```
diff --git a/c/bmgs/fd.c b/c/bmgs/fd.c
index f5e1e2318..c05e425f0 100644
--- a/c/bmgs/fd.c
+++ b/c/bmgs/fd.c
@@ -36,15 +36,16 @@ void *Z(bmgs_fd_worker)(void *threadarg)

     for (int i1 = 0; i1 < s->n[1]; i1++)
       {
+#pragma omp simd
         for (int i2 = 0; i2 < s->n[2]; i2++)
           {
             T x = 0.0;
             for (int c = 0; c < s->ncoefs; c++)
-              x += aa[s->offsets[c]] * s->coefs[c];
-            *bb++ = x;
-            aa++;
+              x += aa[s->offsets[c]+i2] * s->coefs[c];
+            bb[i2] = x;
           }
-        aa += s->j[2];
+        bb += s->n[2];
+        aa += s->j[2] + s->n[2];
       }
   }
   return NULL;
```

Similar code modifications were also done in one of the other potential
targets (`symmetrize()` in `c/symmetry.c`), which had the same kind of overall
structure as the previous two, but with only the position of the input array
being advanced with pointer incrementation. Pointer incrementation was
completely replaced with array indeces and an OpenMP SIMD pragma was added
around the outermost loop to once again give the compiler hints on better
vectorisation.

The full code modifications are available in [omp-simd.diff](./omp-simd.diff).

Intel compiler was confirmed to be able to take advantage of the code
modifications for better vectorisation and the **performance of GPAW was
observed to increase on KNLs by up to 18.8%** (see
[Effects of code modifications](#effects-of-code-modifications) for detailed
results).
Alternative versions of the code modifications (e.g. to test indexing at other
loop levels) were also tried, but with negative or neutral effects.
None of the other potential targets yielded significant performance
increases when using the same kind of optimisation strategy.

After the code modifications, the performance of GPAW seems to be mostly
constrained by a load inbalance that manifests in MPI wait times when doing
global synchronisation. Further work is needed to improve the load balance
e.g. by improving the domain decomposition to have a more uniform distribution
of work between the MPI tasks.

#### Removal of obsolete code

During the code optimisation work, obsolete sections of code were also
identified in the DIIS step of the RMM-DIIS eigensolver.

On Haswell CPUs the unnecessary code consumes only a small fraction of the
computing time for Case 1 (roughly 1% when using a single node, but increasing
with the number of nodes). In contrast, on KNLs the fraction is significantly
higher at approx. 11% of the computing time. Since this time is wasted on
unproductive computing, the removal of these code sections seemed a good way
to not only clean up the code base, but also to improve performance especially
on KNLs.

Unfortunately, the expected performance increase was not achieved on either
architecture with or without the above code modifications. It seems that the
removal of obsolete code sections just exposes more clearly an underlying MPI
inbalance and shifts the saved time to the next MPI synchronisation threshold.

The full code modifications are available in
[diis-error.diff](./diis-error.diff).

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
| - | -----: | ----: | -------: | -------: | -------: | --------: |
| 1 | 329.2  | 330.0 | 319.9    | 320.8    | 321.1    |           |
| 2 | 215.5  | 213.2 | 206.6    |          |          |           |
| 4 |        | 145.5 | 141.3    |          |          |           |
| 8 |        |       | 101.3    | 101.2    | 101.3    | 101.4     |


**Table 2**. Average runtime in seconds for Case 2 when using *n* KNLs. Data
shown for runs using standard memory allocator (*stdlib*) as well as using
tbbmalloc without huge pages (*TBB*) and with 2M, 4M, 8M, or 16M huge pages
(*TBB + 2M* etc.).

| n | stdlib | TBB   | TBB + 2M | TBB + 4M | TBB + 8M |
| - | -----: | ----: | -------: | -------: | -------: |
| 1 | 341.0  | 337.1 | 323.4    | 323.9    | 323.4    |
| 2 |        | 177.7 | 172.3    | 171.0    | 170.9    |
| 4 | 133.0  | 130.7 | 127.0    | 127.0    | 127.2    |
| 8 |        |  82.2 |  80.0    |  80.0    |  80.2    |

Additionally, for Case 1 the effect of using ScaLAPACK instead of serial
LAPACK (which is the default in Case 1) was tested, but only degrading
performance was achieved.

### Performance comparison

Since the size of the huge pages does not affect performance, results for
runs using tbbmalloc together with 2M huge pages were chosen for a
comparison with Haswell CPUs (Table 3).
Please see [runtime-vanilla.data](./runtime-vanilla.data) for more detailed
results, including comparison to other architectures.

**Table 3**. Comparison of runtimes on Haswell CPU nodes (*CPU*) and KNLs
(*KNL*) for both *Case 1* and *Case 2*. Average runtimes in seconds are
shown for 1, 2, 4, or 8 nodes consisting of two CPUs or a single KNL.

|            |     | 1     | 2     | 4     | 8     |
| ---------- | --- | ----: | ----: | ----: | ----: |
| **Case 1** | CPU | 242.2 | 148.5 |  81.1 |  55.4 |
|            | KNL | 319.9 | 206.6 | 141.3 | 101.3 |
| **Case 2** | CPU | 405.0 | 191.5 |  86.9 |  60.5 |
|            | KNL | 323.4 | 172.3 | 127.0 |  80.0 |


**Table 4**. Relative performance of KNLs compared to Haswell CPU nodes
(*CPU / KNL*) for both *Case 1* and *Case 2*. Runtime on CPUs divided by
runtime on KNLs shown for 1, 2, 4, or 8 nodes consisting of two CPUs or a
single KNL.

|            |           | 1     | 2     | 4     | 8     |
| ---------- | --------- | ----: | ----: | ----: | ----: |
| **Case 1** | CPU / KNL | 0.757 | 0.719 | 0.574 | 0.547 |
| **Case 2** | CPU / KNL | 1.252 | 1.111 | 0.684 | 0.756 |

As can been seen from Table 4, for Case 1 the performance of a KNL is at
best 75.7% of that of a Haswell CPU node. In contrast, for Case 2 the
performance of a single KNL is 125.2% of that of a Haswell CPU node. When
using more KNLs (or CPU nodes), it is clear that KNLs do not scale as well
as CPUs and thus already with 4 nodes/KNLs also Case 2 is faster on CPUs
(Table 4).

Nevertheless, the results are quite comparable and depending on the system of
interest KNLs may offer similar (or even better) performance than Haswell
CPUs already out of the box without any code modifications in GPAW.

### Effects of code modifications

By adding OpenMP SIMD pragmas and using explicit array indexing instead of
pointer incrementation in some of the computational kernels (see
[above](#improved-vectorisation-with-openmp-simds) for details), the compiler
was able to vectorise the kernels better for KNLs.
Please see [runtime-optimised.data](./runtime-optimised.data) for more
detailed results.

**Table 5**. Comparison of KNL runtimes before (*vanilla*) and after
(*optimised*) code modifications for both *Case 1* and *Case 2*. Average
runtimes in seconds and the achieved performance increase (*speed-up*) are
shown for 1, 2, 4, or 8 KNLs.

|            |            | 1       | 2       | 4       | 8       |
| ---------- | ---------- | ------: | ------: | ------: | ------: |
| **Case 1** | vanilla    | 319.9   | 206.6   | 141.3   | 101.3   |
|            | optimised  | 269.3   | 177.3   | 123.2   |  91.3   |
|            | *speed-up* | *1.188* | *1.165* | *1.147* | *1.110* |
| **Case 2** | vanilla    | 323.4   | 172.3   | 127.0   |  80.0   |
|            | optimised  | 280.7   | 150.0   | 116.1   |  74.0   |
|            | *speed-up* | *1.152* | *1.149* | *1.094* | *1.081* |


**Table 6**. Relative performance of KNLs compared to Haswell CPU nodes
before (*vanilla*) and after (*optimised*) code modifications for both
*Case 1* and *Case 2*. No code modifications were used on CPUs. Runtime on
CPUs divided by runtime on KNLs shown for 1, 2, 4, or 8 nodes consisting of
two CPUs or a single KNL.

|            |           | 1     | 2     | 4     | 8     |
| ---------- | --------- | ----: | ----: | ----: | ----: |
| **Case 1** | vanilla   | 0.757 | 0.719 | 0.574 | 0.547 |
|            | optimised | 0.899 | 0.838 | 0.658 | 0.607 |
| **Case 2** | vanilla   | 1.252 | 1.111 | 0.684 | 0.756 |
|            | optimised | 1.443 | 1.277 | 0.748 | 0.818 |

As can be seen from Table 5, on a single KNL the runtime dropped by 50.6s
to 269.3s for Case 1 and by 42.7s to 280.7s for Case 2. This speed-up gained
by code optimisation translates to a maximum performance increase of 18.8%
and 15.2% for Case 1 and Case 2, respectively. Using multiple KNLs decreases
the relative speed-up, but even with 8 KNLs the performance increase is more
than 8% even for Case 2.

When looking at the relative performance compared to Haswell CPUs (Table 6),
the overall picture does not change from earlier conclusions. Slightly better
performance is achieved on KNLs, but going beyond 2 nodes CPUs offer better
performance than KNLs on both benchmarks.

## Conclusions

GPAW has been shown to achieve similar performance on KNLs as on dual-CPU
Haswell nodes, but with poorer scaling properties. Depending on the benchmark
the performance is either slightly worse or better (from 75.7% to 125.2% for
a single KNL/node). Unsurprisingly, the higher the computational burden is,
the better KNL performs and scales when moving to multiple processors.

Based on the two benchmarks tested, it is recommended for optimal performance
to use KNLs for workloads similar to Case 2 (Copper filament) if one is
limited to using one or two nodes.

