# Ref: https://github.com/scipy/scipy/blob/main/scipy/meson.build
# commit:  053a3e8
#
# NOTES:
# - As scikit-misc does not use these librares/languages
#   - c++
#   - f2py (https://numpy.org/doc/stable/f2py/),
#   - pythran
#   - pybind


#### Setup NumPy
# Include numpy directory - needed in all submodules
incdir_numpy = run_command(py,
  [
    '-c',
    'import os; os.chdir(".."); import numpy; print(numpy.get_include())'
  ],
  check: true
).stdout().strip()
inc_np = include_directories(incdir_numpy)

npymath_path = incdir_numpy / '..' / 'lib'
npymath_lib = cc.find_library('npymath', dirs: npymath_path)
npyrandom_path = incdir_numpy / '..' / '..' / 'random' / 'lib'
# Note: `required: false` can be removed once numpy 1.19 is the minimum version
npyrandom_lib = cc.find_library('npyrandom', dirs: npyrandom_path, required: false)

# Don't use the deprecated NumPy C API. Define this to a fixed version instead of
# NPY_API_VERSION in order not to break compilation for released SciPy versions
# when NumPy introduces a new deprecation. Use in a meson.build file
#
#   py.extension_module('_name',
#     'source_fname',
#     numpy_nodepr_api)

# numpy_nodepr_api = '-DNPY_NO_DEPRECATED_API=NPY_1_9_API_VERSION'
numpy_nodepr_api = '-DNPY_NO_DEPRECATED_API=1'
###

#### Check Supported Compiler Flags
# This is necessary to ensure that SciPy
# can be built with any supported compiler. We need so many warning flags
# because we want to be able to build with `-Werror` in CI; that ensures that
# for new code we add, there are no unexpected new issues introduced.
#
# Cleaning up code so we no longer need some of these warning flags is useful,
# but not a priority.
#
# The standard convention used here is:
#   - for C, drop the leading dash and turn remaining dashes into underscores
#   - for C++, prepend `_cpp` and turn remaining dashes into underscores
#   - for Fortran, prepend `_fflags` and turn remaining dashes into underscores

# C warning flags
Wno_maybe_uninitialized = cc.get_supported_arguments('-Wno-maybe-uninitialized')
Wno_discarded_qualifiers = cc.get_supported_arguments('-Wno-discarded-qualifiers')
Wno_empty_body = cc.get_supported_arguments('-Wno-empty-body')
Wno_implicit_function_declaration = cc.get_supported_arguments('-Wno-implicit-function-declaration')
Wno_parentheses = cc.get_supported_arguments('-Wno-parentheses')
Wno_switch = cc.get_supported_arguments('-Wno-switch')
Wno_unused_label = cc.get_supported_arguments('-Wno-unused-label')
Wno_unused_variable = cc.get_supported_arguments('-Wno-unused-variable')

# Fortran warning flags
_fflag_Wno_argument_mismatch = ff.get_supported_arguments('-Wno-argument-mismatch')
_fflag_Wno_conversion = ff.get_supported_arguments('-Wno-conversion')
_fflag_Wno_intrinsic_shadow = ff.get_supported_arguments('-Wno-intrinsic-shadow')
_fflag_Wno_maybe_uninitialized = ff.get_supported_arguments('-Wno-maybe-uninitialized')
_fflag_Wno_surprising = ff.get_supported_arguments('-Wno-surprising')
_fflag_Wno_uninitialized = ff.get_supported_arguments('-Wno-uninitialized')
_fflag_Wno_unused_dummy_argument = ff.get_supported_arguments('-Wno-unused-dummy-argument')
_fflag_Wno_unused_label = ff.get_supported_arguments('-Wno-unused-label')
_fflag_Wno_unused_variable = ff.get_supported_arguments('-Wno-unused-variable')
_fflag_Wno_tabs = ff.get_supported_arguments('-Wno-tabs')

# The default list of warnings to ignore from Fortran code. There is a lot of
# old, vendored code that is very bad and we want to compile it silently (at
# least with GCC and Clang)
fortran_ignore_warnings = ff.get_supported_arguments(
 _fflag_Wno_argument_mismatch,
 _fflag_Wno_conversion,
 _fflag_Wno_maybe_uninitialized,
 _fflag_Wno_unused_dummy_argument,
 _fflag_Wno_unused_label,
 _fflag_Wno_unused_variable,
 _fflag_Wno_tabs,
)

###

#### Check Supported Linker Flags

## C
Wlno_compact_unwind = cc.get_supported_link_arguments('-Wl,-no_compact_unwind')
# TODO: This should not exist long term
# Disable warning about undefined dynamic lookup of symbols. It is harmless.
# https://github.com/python/cpython/issues/97524#issuecomment-1270616173
Wlno_fixup_chains = cc.get_supported_link_arguments('-Wl,-no_fixup_chains')
###

#### Setup on Operating System
if is_windows
  is_mingw = is_windows and cc.get_id() == 'gcc'

  # For mingw-w64, link statically against the UCRT.
  gcc_link_args = ['-lucrt', '-static']
  if is_mingw
    add_project_link_arguments(gcc_link_args, language: ['c'])
    # Force gcc to float64 long doubles for compatibility with MSVC
    # builds, for C only.
    add_project_arguments('-mlong-double-64', language: 'c')
    # Make fprintf("%zd") work (see https://github.com/rgommers/scipy/issues/118)
    add_project_arguments('-D__USE_MINGW_ANSI_STDIO=1', language: ['c'])
    # Manual add of MS_WIN64 macro when not using MSVC.
    # https://bugs.python.org/issue28267
    bitness = run_command('_build_utils/gcc_build_bitness.py').stdout().strip()
    if bitness == '64'
      add_project_arguments('-DMS_WIN64', language: ['c', 'fortran'])
    endif
    # Silence warnings emitted by PyOS_snprintf for (%zd), see
    # https://github.com/rgommers/scipy/issues/118.
    # Use as c_args for extensions containing Cython code
    cython_c_args += ['-Wno-format-extra-args', '-Wno-format']
  endif

  # Using gfortran on windows
  if ff.get_id() == 'gcc'
    add_project_link_arguments(gcc_link_args, language: ['fortran'])
    # Flag needed to work around BLAS and LAPACK Gfortran dependence on
    # undocumented C feature when passing single character string
    # arguments.
    # https://gcc.gnu.org/bugzilla/show_bug.cgi?id=90329
    # https://github.com/wch/r-source/blob/838f9d5a7be08f2a8c08e47bcd28756f5d0aac90/src/gnuwin32/MkRules.rules#L121
    add_project_arguments('-fno-optimize-sibling-calls', language: ['fortran'])
  endif

  # Deal with M_PI & friends; add `use_math_defines` to c_args or cpp_args
  # Cython doesn't always get this right itself (see, e.g., gh-16800), so
  # explicitly add the define as a compiler flag for Cython-generated code.
  use_math_defines = ['-D_USE_MATH_DEFINES']
elif is_macos
  cython_c_link_args += [
    Wlno_compact_unwind,
    Wlno_fixup_chains
  ]
else
  use_math_defines = []
endif
###

#### BLAS and LAPACK
# TODO: 64-bit BLAS and LAPACK
#
# Note that this works as long as BLAS and LAPACK are detected properly via
# pkg-config. By default we look for OpenBLAS, other libraries can be configured via
# `meson configure -Dblas=blas -Dlapack=lapack` (example to build with Netlib
# BLAS and LAPACK).
# For MKL and for auto-detecting one of multiple libs, we'll need a custom
# dependency in Meson (like is done for scalapack) - see
# https://github.com/mesonbuild/meson/issues/2835
blas_name = get_option('blas')
lapack_name = get_option('lapack')
# pkg-config uses a lower-case name while CMake uses a capitalized name, so try
# that too to make the fallback detection with CMake work
if blas_name == 'openblas'
  blas_name = ['openblas', 'OpenBLAS']
endif
if lapack_name == 'openblas'
  lapack_name = ['openblas', 'OpenBLAS']
endif
blas = dependency(blas_name)
lapack = dependency(lapack_name)
###

#### Dependencies Information
dependency_map = {
  'BLAS': blas,
  'LAPACK': lapack,
}


foreach name, dep : dependency_map
  conf_data.set(name + '_NAME', dep.name())
  conf_data.set(name + '_FOUND', dep.found())

  if dep.found()
    conf_data.set(name + '_VERSION', dep.version())
    conf_data.set(name + '_TYPE_NAME', dep.type_name())
    conf_data.set(name + '_INCLUDEDIR', dep.get_variable('includedir', default_value: 'unknown'))
    conf_data.set(name + '_LIBDIR', dep.get_variable('libdir', default_value: 'unknown'))
    conf_data.set(name + '_OPENBLAS_CONFIG', dep.get_variable('openblas_config', default_value: 'unknown'))
    conf_data.set(name + '_PCFILEDIR', dep.get_variable('pcfiledir', default_value: 'unknown'))
  endif

endforeach

###

#### Include Python Sources in this Directory
# Copy the main __init__ to the build dir
python_sources = [
  '__init__.py',
  '_distributor_init.py',
]

py.install_sources(
  python_sources,
  subdir: 'skmisc'
)

#### Build __config__.py
configure_file(
  input: '__config__.py.in',
  output: '__config__.py',
  configuration : conf_data,
  install_dir: skmisc_dir,
)
###

#### Build _version.py
configure_file(
  input: '_version.py.in',
  output: '_version.py',
  configuration: conf_data,
  install_dir: skmisc_dir,
)

# Meson does not inculde any generated files in the source distribution
# We want the generated _version.py so that a versioned build can be
# created from the source dist.
meson.add_dist_script('_build_utils/copy_version_to_dist.py')

###

#### Included sub-packages
subdir('loess')
###
