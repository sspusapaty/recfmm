project('recFMM', ['c', 'fortran'],
        version : '0.1.1',
        license : 'GPL-3.0-or-later',
        meson_version : '>=0.49.0',
        default_options : ['c_std=gnu99',
                           'buildtype=release',
                           'default_library=both',
                           'prefix=' + meson.current_source_dir()])

vso = '0'

# ========== compiler options and dependencies

cc = meson.get_compiler('c')
fc = meson.get_compiler('fortran')

cc_is_icc = cc.get_id().contains('intel')
use_opencilk = not cc_is_icc and cc.has_argument('-fopencilk')
use_cilkplus = not use_opencilk \
               and (cc_is_icc or cc.has_argument('-fcilkplus'))

# HACK GNU gcc 8.x and 9.x still accept `-fcilkplus` even though they do not
# actually support Cilk...
cc_is_gcc_with_cilk = cc.get_id() == 'gcc' and \
                      cc.version().version_compare('>=5.0.0') and \
                      cc.version().version_compare('<8.1.0')
if use_cilkplus and cc.get_id() == 'gcc' and not cc_is_gcc_with_cilk
  warning('The specified GCC version does not support Cilk' +
          ' (even though -fcilkplus does not trigger an error).')
  use_cilkplus = false
endif

if use_opencilk or use_cilkplus
  have_cilk_headers = cc.has_header('cilk/cilk.h') and \
                      cc.has_header('cilk/cilk_api.h') and \
                      cc.has_header('cilk/reducer_min.h') and \
                      cc.has_header('cilk/reducer_max.h') and \
                      cc.has_header('cilk/cilk_c11_threads.h')
  if not have_cilk_headers
    warning('Required Cilk headers not found. Switching to sequential build.')
  endif
endif

if cc_is_icc
  add_project_arguments(['-Dicc'], language : 'c') # enables SIMD reduction
endif

m_dep = cc.find_library('m', required : false)

# ========== parallel vs sequential implementation

flag_parallel = (use_opencilk or use_cilkplus) and have_cilk_headers

if flag_parallel                # ----- parallel build

  # Cilk flags
  if use_opencilk
    cilk_cflags  = ['-fopencilk']
    cilk_ldflags = ['-fopencilk']
  elif use_cilkplus and not cc_is_icc
    cilk_cflags  = ['-fcilkplus']
    cilk_ldflags = ['-lcilkrts']
  else
    cilk_cflags  = []
    cilk_ldflags = []
  endif
  add_project_arguments([cilk_cflags, '-DUSING_CILK'], language : 'c')
  add_project_link_arguments([cilk_ldflags], language : 'c')

  # Intel TBB scalable allocator
  tbb_dir = get_option('tbb_dir')
  if tbb_dir == ''
    tbb_dir = []
  endif
  tbbmalloc_dep = dependency('tbbmalloc', required : false)
  if not tbbmalloc_dep.found()
    tbbmalloc_dep = cc.find_library('tbbmalloc', required : true,
                                    dirs : tbb_dir)
  endif

else                            # ----- sequential build

  warning('Could not verify Cilk support with the specified compiler.' +
          ' Cilk keywords elided.')

  tbbmalloc_dep = dependency('', required : false)

endif

recfmm_dep = [tbbmalloc_dep, m_dep]

# ========== instrumentation tools

if get_option('cilktool') == 'cilksan' # ----- Cilksan
  if not (use_opencilk and have_cilk_headers)
    warning('Cilksan instrumentation specified but not using OpenCilk compiler;' +
            ' skipping...')
  else
    add_project_arguments(['-fsanitize=cilk','-fno-stripmine',
                           '-fno-vectorize','-fno-unroll-loops'], language : 'c')
    add_project_link_arguments(['-fsanitize=cilk'], language : 'c')
    # FIXME [2021-01-25] Meson does not support static-library-only arguments to
    # build targets defined via `library()`.  See also:
    # https://github.com/mesonbuild/meson/issues/3304
    if get_option('default_library') != 'static'
      add_project_link_arguments(['-shared-libasan'], language : 'c')
    endif
  endif
endif

# ========== build targets

# headers
subdir('include')
install_headers(recfmm_hdrs, install_dir : 'include')

# source files
subdir('src')
subdir('test')
subdir('example')

def_lap = ['-DLAPLACE']
def_yuk = ['-DYUKAWA']
def_multicilk = ['-DMULTICILK']
def_normal = ['-DNORMAL']
def_serial_cilk = ['-DSERIAL_CILK']
def_par_cilk = ['-DPAR_CILK']

# recFMM libraries: Laplace & Yukawa
lib_lap = library('adap_laplace', [src_recfmm_laplace],
                  c_args : def_lap + def_normal,
                  dependencies : recfmm_dep,
                  include_directories : [recfmm_include_dir],
                  install : true, install_dir : 'lib',
                  soversion : vso)
lib_yuk = library('adap_yukawa', [src_recfmm_yukawa],
                  c_args : def_yuk + def_normal,
                  dependencies : recfmm_dep,
                  include_directories : [recfmm_include_dir],
                  install : true, install_dir : 'lib',
                  soversion : vso)
lib_lap_multicilk = library('adap_laplace_m', [src_recfmm_laplace],
                  c_args : def_lap + def_multicilk,
                  dependencies : recfmm_dep,
                  include_directories : [recfmm_include_dir],
                  install : true, install_dir : 'lib',
                  soversion : vso)
lib_yuk_multicilk = library('adap_yukawa_m', [src_recfmm_yukawa],
                  c_args : def_yuk + def_multicilk,
                  dependencies : recfmm_dep,
                  include_directories : [recfmm_include_dir],
                  install : true, install_dir : 'lib',
                  soversion : vso)
lib_lap_serialcilk = library('adap_laplace_s', [src_recfmm_laplace],
                  c_args : def_lap + def_serial_cilk,
                  dependencies : recfmm_dep,
                  include_directories : [recfmm_include_dir],
                  install : true, install_dir : 'lib',
                  soversion : vso)
lib_yuk_serialcilk = library('adap_yukawa_s', [src_recfmm_yukawa],
                  c_args : def_yuk + def_serial_cilk,
                  dependencies : recfmm_dep,
                  include_directories : [recfmm_include_dir],
                  install : true, install_dir : 'lib',
                  soversion : vso)
lib_lap_parcilk = library('adap_laplace_p', [src_recfmm_laplace],
                  c_args : def_lap + def_par_cilk,
                  dependencies : recfmm_dep,
                  include_directories : [recfmm_include_dir],
                  install : true, install_dir : 'lib',
                  soversion : vso)
lib_yuk_parcilk = library('adap_yukawa_p', [src_recfmm_yukawa],
                  c_args : def_yuk + def_par_cilk,
                  dependencies : recfmm_dep,
                  include_directories : [recfmm_include_dir],
                  install : true, install_dir : 'lib',
                  soversion : vso)

# test executables
exe_test_lap = executable('recfmm-test-laplace', [src_recfmm_test],
                          c_args : def_lap + def_normal, link_with : lib_lap,
                          dependencies : recfmm_dep,
                          include_directories : [recfmm_include_dir],
                          install_rpath : get_option('prefix') / 'lib',
                          install : true, install_dir : 'test')
exe_test_yuk = executable('recfmm-test-yukawa', [src_recfmm_test],
                          c_args : def_yuk + def_normal, link_with : lib_yuk,
                          dependencies : recfmm_dep,
                          include_directories : [recfmm_include_dir],
                          install_rpath : get_option('prefix') / 'lib',
                          install : true, install_dir : 'test')
exe_test_lap_multicilk = executable('recfmm-test-laplace-multicilk', [src_recfmm_test],
                          c_args : def_lap + def_multicilk, link_with : lib_lap_multicilk,
                          dependencies : recfmm_dep,
                          include_directories : [recfmm_include_dir],
                          install_rpath : get_option('prefix') / 'lib',
                          install : true, install_dir : 'test')
exe_test_yuk_multicilk = executable('recfmm-test-yukawa-multicilk', [src_recfmm_test],
                          c_args : def_yuk + def_multicilk, link_with : lib_yuk_multicilk,
                          dependencies : recfmm_dep,
                          include_directories : [recfmm_include_dir],
                          install_rpath : get_option('prefix') / 'lib',
                          install : true, install_dir : 'test')
exe_test_lap_serialcilk = executable('recfmm-test-laplace-serialcilk', [src_recfmm_test],
                          c_args : def_lap + def_serial_cilk, link_with : lib_lap_serialcilk,
                          dependencies : recfmm_dep,
                          include_directories : [recfmm_include_dir],
                          install_rpath : get_option('prefix') / 'lib',
                          install : true, install_dir : 'test')
exe_test_yuk_serialcilk = executable('recfmm-test-yukawa-serialcilk', [src_recfmm_test],
                          c_args : def_yuk + def_serial_cilk, link_with : lib_yuk_serialcilk,
                          dependencies : recfmm_dep,
                          include_directories : [recfmm_include_dir],
                          install_rpath : get_option('prefix') / 'lib',
                          install : true, install_dir : 'test')
exe_test_lap_parcilk = executable('recfmm-test-laplace-parcilk', [src_recfmm_test],
                          c_args : def_lap + def_par_cilk, link_with : lib_lap_parcilk,
                          dependencies : recfmm_dep,
                          include_directories : [recfmm_include_dir],
                          install_rpath : get_option('prefix') / 'lib',
                          install : true, install_dir : 'test')
exe_test_yuk_parcilk = executable('recfmm-test-yukawa-parcilk', [src_recfmm_test],
                          c_args : def_yuk + def_par_cilk, link_with : lib_yuk_parcilk,
                          dependencies : recfmm_dep,
                          include_directories : [recfmm_include_dir],
                          install_rpath : get_option('prefix') / 'lib',
                          install : true, install_dir : 'test')

# example executable
exe_example_lap = executable('recfmm-example', [src_recfmm_example],
                             c_args : def_lap, link_with : lib_lap,
                             dependencies : recfmm_dep,
                             include_directories : [recfmm_include_dir],
                             install_rpath : get_option('prefix') / 'lib',
                             install : true, install_dir : 'example')

# ========== tests

test_verifier = find_program('test/verify_test_pass.sh')

test('Laplace [box, acc=3]', test_verifier,
     args : [exe_test_lap, '-d1', '-a3'],
     suite : ['laplace', 'box', 'low_acc'])
test('Laplace [sphere, acc=3]', test_verifier,
     args : [exe_test_lap, '-d2', '-a3'],
     suite : ['laplace', 'sphere', 'low_acc'])
test('Laplace [box, acc=6]', test_verifier,
     args : [exe_test_lap, '-d1', '-a6'],
     suite : ['laplace', 'box', 'high_acc'])
test('Laplace [sphere, acc=6]', test_verifier,
     args : [exe_test_lap, '-d2', '-a6'],
     suite : ['laplace', 'sphere', 'high_acc'])

test('Yukawa [box, acc=3]', test_verifier,
     args : [exe_test_yuk, '-d1', '-a3'],
     suite : ['yukawa', 'box', 'low_acc'])
test('Yukawa [sphere, acc=3]', test_verifier,
     args : [exe_test_yuk, '-d2', '-a3'],
     suite : ['yukawa', 'sphere', 'low_acc'])
test('Yukawa [box, acc=6]', test_verifier,
     args : [exe_test_yuk, '-d1', '-a6'],
     suite : ['yukawa', 'box', 'high_acc'])
test('Yukawa [sphere, acc=6]', test_verifier,
     args : [exe_test_yuk, '-d2', '-a6'],
     suite : ['yukawa', 'sphere', 'high_acc'])
