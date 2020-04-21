project('lc0', 'cpp',
        default_options : ['cpp_std=c++17', 'b_ndebug=if-release', 'warning_level=3'],
        meson_version: '>=0.52')



cc = meson.get_compiler('emscription')

# Files to compile.
deps = []
files = []
includes = []
has_backends = false

# Third party files.
includes += include_directories('third_party', is_system: true)

# Compiling protobufs.
compile_proto = find_program('scripts/compile_proto.py')
gen = generator(compile_proto, output: ['@BASENAME@.pb.h'],
  arguments : [
    '--proto_path=@CURRENT_SOURCE_DIR@/libs/lczero-common',
    '--cpp_out=@BUILD_DIR@',
    '@INPUT@'])

# Handle submodules.
git = find_program('git', required: false)
if run_command('scripts/checkdir.py', 'libs/lczero-common/proto').returncode() != 0
  if git.found()
    if run_command(git, 'status').returncode() == 0
      message('updating git submodule libs/lczero-common')
      run_command(git, 'submodule', 'update', '--init', '--recursive')
    else
      message('cloning lczero-common.git into libs/lczero-common')
      run_command(git, 'clone', '--depth=1',
                  'https://github.com/LeelaChessZero/lczero-common.git',
                  'libs/lczero-common/')
    endif
  else
    error('Please install git to automatically fetch submodules or download the archives manually from GitHub.')
  endif
endif

pb_files = [
  'src/utils/protomessage.cc',
  gen.process('libs/lczero-common/proto/net.proto',
    preserve_path_from : meson.current_source_dir() + '/libs/lczero-common/')
]
files += pb_files

# Extract git short revision.
short_rev = 'unknown'
if git.found()
  r = run_command(git, 'rev-parse', '--short', 'HEAD')
  if r.returncode() == 0
    # Now let's check if the working directory is clean.
    if run_command(git, 'diff-index', '--quiet', 'HEAD').returncode() == 0
      short_rev = r.stdout().strip()
    else
      short_rev = 'dirty'
      warning('Cannot extract valid git short revision from dirty working directory.')
    endif
  else
    warning('Failed to parse short revision. Use git clone instead of downloading the archive from GitHub.')
  endif
endif

# Construct build identifier.
build_identifier = 'git.' + short_rev
add_project_arguments('-DBUILD_IDENTIFIER="' + build_identifier + '"', language : 'cpp')
message('Using build identifier "' + build_identifier + '".')

#############################################################################
## Main files
#############################################################################
files += [
  'src/engine.cc',
  'src/version.cc',
  'src/benchmark/backendbench.cc',
  'src/benchmark/benchmark.cc',
  'src/chess/bitboard.cc',
  'src/chess/board.cc',
  'src/chess/position.cc',
  'src/chess/uciloop.cc',
  'src/mcts/node.cc',
  'src/mcts/params.cc',
  'src/mcts/search.cc',
  'src/mcts/stoppers/common.cc',
  'src/mcts/stoppers/factory.cc',
  'src/mcts/stoppers/legacy.cc',
  'src/mcts/stoppers/stoppers.cc',
  'src/mcts/stoppers/timemgr.cc',
  'src/neural/cache.cc',
  'src/neural/encoder.cc',
  'src/neural/factory.cc',
  'src/neural/loader.cc',
  'src/neural/network_check.cc',
  'src/neural/network_demux.cc',
  'src/neural/network_legacy.cc',
  'src/neural/network_mux.cc',
  'src/neural/network_random.cc',
  'src/neural/network_rr.cc',
  'src/neural/writer.cc',
  'src/selfplay/game.cc',
  'src/selfplay/loop.cc',
  'src/selfplay/tournament.cc',
  'src/syzygy/syzygy.cc',
  'src/utils/commandline.cc',
  'src/utils/configfile.cc',
  'src/utils/esc_codes.cc',
  'src/utils/histogram.cc',
  'src/utils/logging.cc',
  'src/utils/optionsdict.cc',
  'src/utils/optionsparser.cc',
  'src/utils/random.cc',
  'src/utils/string.cc',
  'src/utils/weights_adapter.cc',
]
includes += include_directories('src')

deps += dependency('threads')

if get_option('build_backends')
  ## ~~~~~~~~~~
  ## Tensorflow
  ## ~~~~~~~~~~
  tf_dl_lib = cc.find_library('dl', required: false)
  tf_tensorflow_cc_lib = dependency('tensorflow_cc',
                                    required: false, include_type:'system')
  if get_option('tensorflow') and tf_dl_lib.found() and tf_tensorflow_cc_lib.found()
    deps += [tf_dl_lib, tf_tensorflow_cc_lib]
    files += 'src/neural/network_tf_cc.cc'
    has_backends = true
  endif

  ## ~~~~~
  ## Blas
  ## ~~~~~

  shared_files = []

  accelerate_lib = dependency('Accelerate', required: false)

  mkl_libdirs = get_option('mkl_libdirs')
  mkl_lib = cc.find_library('mkl_rt', dirs: mkl_libdirs, required: false)
  if not mkl_lib.found()
    mkl_lib = cc.find_library('mklml', dirs: mkl_libdirs, required: false)
  endif

  dnnl_libdirs = get_option('dnnl_dir') + '/lib'
  dnnl_lib = cc.find_library('dnnl', dirs: dnnl_libdirs, required: false)

  openblas_libdirs = get_option('openblas_libdirs')
  openblas_lib = cc.find_library('openblas.dll', dirs: openblas_libdirs, required: false)
  if not openblas_lib.found()
    openblas_lib = cc.find_library('openblas', dirs: openblas_libdirs, required: false)
  endif

  if get_option('blas')
    if get_option('mkl') and mkl_lib.found()
      add_project_arguments(['-DUSE_MKL', '-DUSE_BLAS'], language : 'cpp')
      includes += include_directories(get_option('mkl_include'))
      deps += [ mkl_lib ]

    elif get_option('dnnl') and dnnl_lib.found()
      add_project_arguments(['-DUSE_DNNL', '-DUSE_BLAS'], language : 'cpp')
      includes += include_directories(get_option('dnnl_dir') + '/include')
      deps += [ dnnl_lib, dependency('openmp', required:true) ]

    elif get_option('accelerate') and accelerate_lib.found()
      deps += [ accelerate_lib ]
      add_project_arguments('-DUSE_BLAS', language : 'cpp')

    elif get_option('openblas') and openblas_lib.found()
      add_project_arguments(['-DUSE_OPENBLAS', '-DUSE_BLAS'], language : 'cpp')

      required_openblas_header = 'openblas_config.h'
      if not cc.has_header(required_openblas_header)
        openblas_headers_found = false

        # add the first valid include directory
        foreach d : get_option('openblas_include')
          if not openblas_headers_found and cc.has_header(required_openblas_header, args: '-I' + d)
            includes += include_directories(d)
            openblas_headers_found = true
          endif
        endforeach

        if not openblas_headers_found
          error('Failed to detect OpenBLAS headers. Did you install libopenblas-dev?')
        endif
      endif

      deps += [ openblas_lib ]

    endif

    deps += subproject('eigen').get_variable('eigen_dep')

    blas_files = [
    'src/neural/blas/convolution1.cc',
    'src/neural/blas/fully_connected_layer.cc',
    'src/neural/blas/se_unit.cc',
    'src/neural/blas/network_blas.cc',
    'src/neural/blas/winograd_convolution3.cc'
    ]

    shared_files = [
    'src/neural/shared/activation.cc',
    'src/neural/shared/winograd_filter.cc',
    ]

    files += blas_files
    has_backends = true

    has_opencl = false

  opencl_libdirs = get_option('opencl_libdirs')
  opencl_lib=cc.find_library('OpenCL', dirs: opencl_libdirs, required: false)

  opencl_framework=dependency('OpenCL', method: 'extraframework', required: false)
  if opencl_framework.found()
      opencl_dep = [ opencl_framework ]
      has_opencl = true

  elif opencl_lib.found() and cc.has_header('CL/opencl.h', args: '-I' + get_option('opencl_include'))
      opencl_dep = [ opencl_lib ]
      has_opencl = true

  endif

  if get_option('opencl') and has_opencl

    opencl_files = [
      'src/neural/opencl/network_opencl.cc',
      'src/neural/opencl/OpenCL.cc',
      'src/neural/opencl/OpenCLTuner.cc',
      'src/neural/opencl/OpenCLBuffers.cc',
    ]

    shared_files = [
    'src/neural/shared/activation.cc',
    'src/neural/shared/winograd_filter.cc',
    ]

    if not opencl_framework.found()
      includes += include_directories(get_option('opencl_include'))
    endif
    deps += opencl_dep
    files += opencl_files
    has_backends = true

  endif

  files += shared_files
  

#############################################################################
## Dependencies
#############################################################################
  ## ~~~~
  ## zlib
  ## ~~~~
  # Pick latest from https://wrapdb.mesonbuild.com/zlib and put into
  # subprojects/zlib.wrap
  deps += dependency('zlib', fallback: ['zlib', 'zlib_dep'])

  ## ~~~~~~~~
  ## Profiler
  ## ~~~~~~~~
  if get_option('buildtype') != 'release'
    deps += cc.find_library('libprofiler',
      dirs: ['/usr/local/lib'], required: false)
  endif
  endif
  endif