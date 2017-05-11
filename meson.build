project('frida-gum', 'c', 'cpp', version: '1.0.0')

languages = ['c', 'cpp']
if host_machine.system() == 'darwin'
  languages += ['objc', 'objcpp']
  add_languages('objc', 'objcpp')
endif

gum_version = meson.project_version()
api_version = '1.0'

cc = meson.get_compiler('c')

cdata = configuration_data()

os_defines = [
  ['windows', 'HAVE_WINDOWS'],
  ['darwin', 'HAVE_DARWIN'],
  ['linux', 'HAVE_LINUX'],
  ['qnx', 'HAVE_QNX'],
]
foreach d : os_defines
  if d.get(0) == host_machine.system()
    cdata.set(d.get(1), 1)
  endif
endforeach

host_os = host_machine.system()

target_conditionals_prefix = '#include <TargetConditionals.h>'

is_macos_src = target_conditionals_prefix + '''
#if !TARGET_OS_MAC
# error Not macOS
#endif
'''
if cc.compiles(is_macos_src, name: 'compiling for macOS')
  host_os = 'macos'
  cdata.set('HAVE_MACOS', 1)
endif

is_ios_src = target_conditionals_prefix + '''
#if !TARGET_OS_IOS
# error Not iOS
#endif
'''
if cc.compiles(is_ios_src, name: 'compiling for iOS')
  host_os = 'ios'
  cdata.set('HAVE_IOS', 1)
endif

if cc.has_header('android/api-level.h')
  host_os = 'android'
  cdata.set('HAVE_ANDROID', 1)
endif

cpu_defines = [
  ['x86', 'HAVE_I386'],
  ['x86_64', 'HAVE_I386'],
  ['arm', 'HAVE_ARM'],
  ['arm64', 'HAVE_ARM64'],
  ['mips', 'HAVE_MIPS'],
]
foreach d : cpu_defines
  if d.get(0) == host_machine.cpu()
    cdata.set(d.get(1), 1)
  endif
endforeach

headers = [
  'elf.h',
  'stdint.h',
  'sys/elf.h',
  'sys/user.h',
]
foreach h : headers
  if cc.has_header(h)
    cdata.set('HAVE_' + h.underscorify().to_upper(), 1)
  endif
endforeach

types = [
  'long double',
  'long long int',
  'unsigned long long int'
]
foreach t : types
  if cc.has_type(t)
    cdata.set('HAVE_' + t.underscorify().to_upper(), 1)
  endif
endforeach

glibc_src = '''
#include <features.h>

#if defined (__GLIBC__) && !defined (__UCLIBC__)
#else
# error Not glibc
#endif
'''
if cc.compiles(glibc_src, name: 'compiling for glibc')
  cdata.set('HAVE_GLIBC', 1)
endif

if cc.has_function('log2')
  cdata.set('HAVE_LOG2', 1)
endif

if cc.has_member('struct mallinfo', 'arena')
  cdata.set('HAVE_LIBC_MALLINFO', 1)
endif

asan_src = '''
#ifndef __SANITIZE_ADDRESS__
# ifdef __clang__
#  if __has_feature(address_sanitizer)
#   define __SANITIZE_ADDRESS__
#  endif
# endif
#endif
#ifndef __SANITIZE_ADDRESS__
# error ASAN disabled
#endif
'''
if cc.compiles(asan_src, name: 'compiling with ASan')
  cdata.set('HAVE_ASAN', 1)
endif

glib_dep = dependency('glib-2.0')
gobject_dep = dependency('gobject-2.0')
gmodule_dep = dependency('gmodule-2.0')
gio_dep = dependency('gio-2.0')
capstone_dep = dependency('capstone')
ffi_dep = dependency('libffi')

extra_deps = []
extra_requires_private = []
extra_libraries_private = []

gumjs_extra_deps = []
gumjs_extra_requires = []
gumjs_extra_libraries = []

unwind_dep = dependency('libunwind', required: false)
unwind_requires = []
if unwind_dep.found()
  cdata.set('HAVE_LIBUNWIND', 1)
  extra_deps += [unwind_dep]
  extra_requires_private += ['libunwind']
elif host_machine.system() == 'linux' or host_machine.system() == 'qnx'
  error('libunwind is required')
endif

have_bfd = cc.has_header('bfd.h')
if have_bfd
  if host_machine.system() == 'linux'
    extra_libraries_private += ['-lbfd', '-ldl', '-lz']
  else
    extra_libraries_private += ['-lbfd', '-lz']
  endif
elif host_machine.system() == 'linux' or host_machine.system() == 'qnx'
  error('libbfd is required')
endif

if get_option('enable_gumpp')
  cdata.set('HAVE_GUMPP', 1)
endif

if get_option('enable_gumjs')
  cdata.set('HAVE_GUMJS', 1)

  json_glib_dep = dependency('json-glib-1.0')

  if host_machine.system() == 'windows'
    gio_os_package_name = 'gio-windows-2.0'
  else
    gio_os_package_name = 'gio-unix-2.0'
  endif
  gio_os_package_dep = dependency(gio_os_package_name)
  gumjs_extra_deps += [gio_os_package_dep]
  gumjs_extra_requires += [gio_os_package_name]

  if host_machine.system() == 'linux' or host_machine.system() == 'qnx'
    gumjs_extra_libraries += ['-lm']
  endif

  if not get_option('enable_diet')
    v8_dep = dependency('v8', version: '>=6.0.124', required: false)
    have_v8 = v8_dep.found()
  else
    have_v8 = false
  endif

  if have_v8
    cdata.set('HAVE_V8', 1)
    gumjs_extra_deps += [v8_dep]
    gumjs_extra_requires += ['v8']
  endif
endif

configure_file(input: 'config.h.meson',
  output: 'config.h',
  configuration: cdata)

add_project_arguments('-include', 'config.h', language: languages)

if host_machine.system() == 'linux'
  add_project_arguments('-D_GNU_SOURCE=1', language: languages)
endif

gum_incdirs = [
  include_directories('.'),
  include_directories('gum'),
  include_directories('gum/arch-x86'),
  include_directories('gum/arch-arm'),
  include_directories('gum/arch-arm64'),
  include_directories('gum/arch-mips'),
  include_directories('libs'),
  include_directories('libs/gum/heap'),
  include_directories('libs/gum/prof'),
]

install_header_basedir = 'frida-@0@'.format(api_version)
install_header_subdir = install_header_basedir + '/gum'

subdir('gum')
subdir('libs')
subdir('bindings')
subdir('vapi')
subdir('tests')