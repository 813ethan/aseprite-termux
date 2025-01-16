# aseprite-termux
This guide will show you how to build the animated sprite and pixel art tool [aseprite](https://www.aseprite.org/) for termux-x11.

The building process requires around 8GB of empty storage.

**NOTE:** **Android 8 or higher** is required in order to build and run aseprite (`pthread_getname_np` is needed and only available after Android 8)

repo branches used in this guide:

- Aseprite: `main` [`v1.3.11`](https://github.com/aseprite/aseprite/tree/v1.3.11)
- Skia: `aseprite-m102` [`857111b`](https://github.com/aseprite/skia/tree/857111b32703e9e8cb14053e25f56b21d636a3c2)

# Steps

**1. install essential packages for compilation**
```
pkg install x11-repo
```
```
pkg install build-essential binutils-is-llvm gn libglvnd* libsm libx11 libxcursor libxi python xdg-utils xorgproto
```

**2. clone skia and aseperite**
```
cd $HOME
git clone --recursive https://github.com/aseprite/aseprite.git
```
```
mkdir $HOME/deps
cd $HOME/deps
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
git clone -b aseprite-m102 https://github.com/aseprite/skia.git
cd skia
python tools/git-sync-deps
```

**3. apply patches required to build**

> **How to apply patches?**
> 
> 1. copy and paste all patches into a file in $HOME
> 
> 2. `patch -p0 < file`

aseprite/third_party/libwebp/CMakeLists.txt
```diff
--- aseprite/third_party/libwebp/CMakeLists.txt-old  2024-11-12 07:52:13.306967153 +1100
+++ aseprite/third_party/libwebp/CMakeLists.txt      2024-11-11 08:03:56.817791250 +1100
@@ -100,16 +100,16 @@
 # ##############################################################################
 # Android only.
 if(ANDROID)
-  include_directories(${ANDROID_NDK}/sources/android/cpufeatures)
-  add_library(cpufeatures STATIC
-              ${ANDROID_NDK}/sources/android/cpufeatures/cpu-features.c)
-  list(APPEND INSTALLED_LIBRARIES cpufeatures)
-  target_link_libraries(cpufeatures dl)
-  set(WEBP_DEP_LIBRARIES ${WEBP_DEP_LIBRARIES} cpufeatures)
-  set(WEBP_DEP_INCLUDE_DIRS ${WEBP_DEP_INCLUDE_DIRS}
-      ${ANDROID_NDK}/sources/android/cpufeatures)
-  add_definitions(-DHAVE_CPU_FEATURES_H=1)
-  set(HAVE_CPU_FEATURES_H 1)
+#  include_directories(${ANDROID_NDK}/sources/android/cpufeatures)
+#  add_library(cpufeatures STATIC
+#              ${ANDROID_NDK}/sources/android/cpufeatures/cpu-features.c)
+#  list(APPEND INSTALLED_LIBRARIES cpufeatures)
+#  target_link_libraries(cpufeatures dl)
+#  set(WEBP_DEP_LIBRARIES ${WEBP_DEP_LIBRARIES} cpufeatures)
+#  set(WEBP_DEP_INCLUDE_DIRS ${WEBP_DEP_INCLUDE_DIRS}
+#      ${ANDROID_NDK}/sources/android/cpufeatures)
+#  add_definitions(-DHAVE_CPU_FEATURES_H=1)
+#  set(HAVE_CPU_FEATURES_H 1)
 else()
   set(HAVE_CPU_FEATURES_H 0)
 endif()
```

aseprite/laf/base/memory.cpp
```diff
--- aseprite/laf/base/memory.cpp-old      2024-11-12 01:34:30.572760696 +1100
+++ aseprite/laf/base/memory.cpp  2024-11-11 21:59:51.787887706 +1100
@@ -303,6 +303,13 @@

 #endif

+static void *_aligned_alloc(size_t align, size_t size)
+{
+       void *result = NULL;
+       posix_memalign(&result, align, size);
+       return result;
+}
+
 void* base_aligned_alloc(std::size_t bytes, std::size_t alignment)
 {
 #if LAF_WINDOWS
@@ -312,7 +319,7 @@
   std::size_t misaligned = (bytes % alignment);
   if (misaligned > 0)
     bytes += alignment - misaligned;
-  return aligned_alloc(alignment, bytes);
+  return _aligned_alloc(alignment, bytes);
 #endif
 }
```

aseprite/third_party/harfbuzz/src/hb.hh
```diff
--- aseprite/third_party/harfbuzz/src/hb.hh-old   2024-11-12 01:47:10.084760406 +1100
+++ aseprite/third_party/harfbuzz/src/hb.hh       2024-11-11 20:49:21.005342346 +1100
@@ -125,6 +125,7 @@

 /* Ignored intentionally. */
 #ifndef HB_NO_PRAGMA_GCC_DIAGNOSTIC_IGNORED
+#pragma GCC diagnostic ignored "-Wcast-function-type-strict"
 #pragma GCC diagnostic ignored "-Wclass-memaccess"
 #pragma GCC diagnostic ignored "-Wformat-nonliteral"
 #pragma GCC diagnostic ignored "-Wformat-zero-length"
```

aseprite/laf/base/thread.cpp
```diff
--- aseprite/laf/base/thread.cpp-old
+++ aseprite/laf/base/thread.cpp
@@ -113,6 +113,8 @@
 #endif
 }
 
+extern "C" int pthread_getname_np(pthread_t __pthread, char* _Nonnull __buf, size_t __n);
+
 std::string this_thread::get_name()
 {
 #if LAF_WINDOWS
```

deps/skia/src/ports/SkDebug_stdio.cpp
```diff
--- deps/skia/src/ports/SkDebug_stdio.cpp-old
+++ deps/skia/src/ports/SkDebug_stdio.cpp
@@ -6,7 +6,7 @@
  */

 #include "include/core/SkTypes.h"
-#if !defined(SK_BUILD_FOR_WIN) && !defined(SK_BUILD_FOR_ANDROID)
+//#if !defined(SK_BUILD_FOR_WIN) && !defined(SK_BUILD_FOR_ANDROID)

 #include <stdarg.h>
 #include <stdio.h>
@@ -17,4 +17,4 @@
     vfprintf(stderr, format, args);
     va_end(args);
 }
-#endif//!defined(SK_BUILD_FOR_WIN) && !defined(SK_BUILD_FOR_ANDROID)
+//#endif//!defined(SK_BUILD_FOR_WIN) && !defined(SK_BUILD_FOR_ANDROID)
```

aseprite/laf/base/launcher.cpp (for desktop xdg-open)
```diff
--- aseprite/laf/base/launcher.cpp-old
+++ aseprite/laf/base/launcher.cpp
@@ -81,7 +81,7 @@
   #if __APPLE__
   ret = std::system(("open \"" + file + "\"").c_str());
   #else
-  ret = std::system(("setsid xdg-open \"" + file + "\"").c_str());
+  ret = std::system(("setsid xdg-utils-xdg-open \"" + file + "\"").c_str());
   #endif
 
 #endif
@@ -123,7 +123,7 @@
   if (!base::is_directory(file))
     file = base::get_file_path(file);
 
-  const int ret = std::system(("setsid xdg-open \"" + file + "\"").c_str());
+  const int ret = std::system(("setsid xdg-utils-xdg-open \"" + file + "\"").c_str());
   return (ret == 0);
 
   #endif

```

optional: patch for termux prefix paths (permission denials might happen if not applied)
```diff
--- aseprite/laf/base/fs_unix.h-old
+++ aseprite/laf/base/fs_unix.h
@@ -188,7 +188,7 @@
   char* tmpdir = getenv("TMPDIR");
   if (tmpdir)
     return tmpdir;
-  return "/tmp";
+  return "/data/data/com.termux/files/usr/tmp";
 }
 
 std::string get_user_docs_folder()

--- aseprite/third_party/jpeg/jmemname.c-old
+++ aseprite/third_party/jpeg/jmemname.c
@@ -67,7 +67,7 @@
  */
 
 #ifndef TEMP_DIRECTORY          /* can override from jconfig.h or Makefile */
-#define TEMP_DIRECTORY  "/usr/tmp/" /* recommended setting for Unix */
+#define TEMP_DIRECTORY  "/data/data/com.termux/files/usr/tmp/" /* recommended setting for Unix */
 #endif
 
 static int next_file_num;       /* to distinguish among several temp files */

--- aseprite/third_party/libarchive/libarchive/archive_util.c-old
+++ aseprite/third_party/libarchive/libarchive/archive_util.c
@@ -404,7 +404,7 @@
 #ifdef _PATH_TMP
 		tmp = _PATH_TMP;
 #else
-                tmp = "/tmp";
+                tmp = "/data/data/com.termux/files/usr/tmp";
 #endif
 	archive_strcpy(temppath, tmp);
 	if (temppath->s[temppath->length-1] != '/')

--- aseprite/third_party/lua/loslib.c-old
+++ aseprite/third_party/lua/loslib.c
@@ -105,10 +105,10 @@
 
 #include <unistd.h>
 
-#define LUA_TMPNAMBUFSIZE	32
+#define LUA_TMPNAMBUFSIZE	256
 
 #if !defined(LUA_TMPNAMTEMPLATE)
-#define LUA_TMPNAMTEMPLATE	"/tmp/lua_XXXXXX"
+#define LUA_TMPNAMTEMPLATE     "/data/data/com.termux/files/usr/tmp/lua_XXXXXX"
 #endif

 #define lua_tmpnam(b,e) { \

--- aseprite/third_party/lua/luaconf.h-old
+++ aseprite/third_party/lua/luaconf.h
@@ -223,7 +223,7 @@
 
 #else			/* }{ */
 
-#define LUA_ROOT	"/usr/local/"
+#define LUA_ROOT	"/data/data/com.termux/files/usr/"
 #define LUA_LDIR	LUA_ROOT "share/lua/" LUA_VDIR "/"
 #define LUA_CDIR	LUA_ROOT "lib/lua/" LUA_VDIR "/"

--- aseprite/src/app/font_path_unix.cpp-old
+++ aseprite/src/app/font_path_unix.cpp
@@ -27,8 +27,7 @@
 
   std::queue<std::string> q;
   q.push("~/.fonts");
-  q.push("/usr/local/share/fonts");
-  q.push("/usr/share/fonts");
+  q.push("/data/data/com.termux/files/usr/share/fonts");
 
   while (!q.empty()) {
     std::string fontDir = q.front();

--- deps/skia/src/core/SkVM.cpp-old
+++ deps/skia/src/core/SkVM.cpp
@@ -3054,7 +3054,7 @@
         SkASSERT(false == llvm::verifyModule(*mod, &llvm::outs()));
 
         if (true) {
-            SkString path = SkStringPrintf("/tmp/%s.bc", debug_name);
+            SkString path = SkStringPrintf("/data/data/com.termux/files/tmp/%s.bc", debug_name);
             std::error_code err;
             llvm::raw_fd_ostream os(path.c_str(), err);
             if (err) {
```

**4. build skia**
```
cd $HOME/deps/skia
gn gen out/Release-arm64 --args='is_debug=false is_official_build=true skia_use_system_expat=false skia_use_system_icu=false skia_use_system_libjpeg_turbo=false skia_use_system_libpng=false skia_use_system_libwebp=false skia_use_system_zlib=false skia_use_sfntly=false skia_use_freetype=true skia_use_harfbuzz=true skia_pdf_subset_harfbuzz=true skia_use_system_freetype2=false skia_use_system_harfbuzz=false cc="clang" cxx="clang++" extra_cflags_cc=["-stdlib=libc++"] extra_ldflags=["-stdlib=libc++"]'
ninja -C out/Release-arm64 skia modules
```

**5. build aseprite**
```
cd $HOME/aseprite
mkdir build && cd build
cmake \
 -DCMAKE_BUILD_TYPE=RelWithDebInfo \
 -DCMAKE_CXX_FLAGS:STRING=-stdlib=libc++ \
 -DCMAKE_EXE_LINKER_FLAGS:STRING=-stdlib=libc++ \
 -DCMAKE_C_COMPILER=clang \
 -DCMAKE_CXX_COMPILER=clang++ \
 -DLAF_BACKEND=skia \
 -DSKIA_DIR=$HOME/deps/skia \
 -DSKIA_LIBRARY_DIR=$HOME/deps/skia/out/Release-arm64 \
 -DLAF_WITH_EXAMPLES=OFF \
 -DENABLE_DESKTOP_INTEGRATION=ON \
 -DCMAKE_INSTALL_PREFIX:PATH=$PREFIX \
 -G Ninja \
 ..
ninja aseprite
ninja install
```

**6. enjoy!**
```
aseprite
```
