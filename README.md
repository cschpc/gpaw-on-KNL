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

## Conclusions

