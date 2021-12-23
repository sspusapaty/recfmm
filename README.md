# Changes from the original `recfmm` code

I changed the code to now use one of four different "modes", which you can see from the `ifdef`s in `src/fmm-action.c`. The modes are as follows
* NORMAL: the original `recfmm` code with a single `disaggregate` function
* SERIAL_CILK: the calls to `disaggregate_far` and `disaggregate_near` are run in series, but they each use Cilk internally
* PAR_CILK: the calls to `disaggregate_far` and `disaggregate_near` are run in parallel via calls to `cilk_spawn`.
* MULTICILK: the calls to `disaggregate_far` and `disaggregate_near` are run in independent cilks. Their respective Cilk runtimes can be configured with the environment variables `FARCILK` and `NEARCILK`.

I changed the `meson.build` to create 4 test binaries (1 for each of the modes) for each of yukawa and laplace.
