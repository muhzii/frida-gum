runner_sources = [
  'gumtest.c',
  'testutil.c',
  'stubs/dummyclasses.c',
  'stubs/fakebacktracer.c',
  'stubs/fakeeventsink.c',
  'stalkerdummychannel.c',
  'lowlevelhelpers.c',
]

test_incdirs = [
  include_directories('.'),
  include_directories('stubs'),
]

subdir('core')
subdir('heap')
subdir('prof')
if have_gumjs
  subdir('gumjs')
else
  gum_tests_gumjs = []
endif

subdir('data')

runner_name = 'gum-tests'
if host_os_family == 'darwin'
  runner_name = 'gum-tests-unsigned'
endif

extra_test_deps = [lzma_dep]
extra_link_args = []
extra_link_depends = []

if have_gumjs
  extra_test_deps += [gumjs_dep]
endif

force_cpp_linking = have_gumpp or have_v8
if force_cpp_linking
  if host_os_family == 'darwin'
    # Work around Meson bug: https://github.com/mesonbuild/meson/issues/1766
    runner_sources += ['dummy.mm']
  else
    runner_sources += ['dummy.cpp']
  endif
endif

if host_os_family == 'darwin'
  symlist = 'gum-tests.symbols'
  extra_link_args += ['-Wl,-exported_symbols_list,' + join_paths(meson.current_source_dir(), symlist)]
  extra_link_depends += [symlist]
elif host_os_family != 'windows'
  if host_os == 'android'
    symscript = 'gum-tests-android.version'
  else
    symscript = 'gum-tests.version'
  endif
  extra_link_args += ['-Wl,--version-script,' + join_paths(meson.current_source_dir(), symscript)]
  extra_link_depends += [symscript]
endif

if host_os == 'macos'
  extra_link_args += ['-Wl,-framework,AppKit']
endif

if host_os == 'ios'
  extra_link_args += ['-Wl,-framework,Foundation']
endif

runner = executable(runner_name, runner_sources + [test_data_stamp],
  link_with: [
    gum_tests_core,
    gum_tests_heap,
    gum_tests_prof,
    gum_tests_gumjs,
  ],
  dependencies: [gum_dep, gum_heap_dep, gum_prof_dep, tls_provider_dep] + extra_test_deps,
  link_args: extra_link_args,
  link_depends: extra_link_depends,
)

if host_os_family == 'darwin'
  custom_target('gum-tests-signed',
    input: [
      runner,
      'gum-tests.xcent',
    ],
    output: 'gum-tests',
    command: [
      files('sign.sh'),
      host_os,
      '@INPUT@',
      '@OUTPUT@',
    ],
    build_by_default: true,
  )
endif
