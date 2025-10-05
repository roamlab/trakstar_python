cmake_minimum_required(VERSION 2.8)

project(atc3dgtracker)

# Where custom find modules live
set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)

# ------------------ Global RPATH policy (CMake 2.8 compatible) ------------------
# Keep RPATHs; don't strip them.
set(CMAKE_SKIP_BUILD_RPATH FALSE)
set(CMAKE_SKIP_INSTALL_RPATH FALSE)

# Use the *install* rpath while building too, so running from the build tree works
# without extra env vars. (This is the most 2.8-compatible way to set a build RPATH.)
set(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)

# Also include linker search paths automatically in the final RPATH
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

# Compute a conda lib path if available
if(DEFINED ENV{CONDA_PREFIX})
  set(_CONDA_LIB "$ENV{CONDA_PREFIX}/lib")
else()
  set(_CONDA_LIB "")
endif()

# Where to look at runtime:
# - next to the installed binaries (../lib)
# - and your conda env's lib as a fallback (if present)
if(APPLE)
  # macOS uses @loader_path instead of $ORIGIN
  set(CMAKE_INSTALL_RPATH "@loader_path/../lib;${_CONDA_LIB}")
else()
  # Escape $ for CMake
  set(CMAKE_INSTALL_RPATH "\$ORIGIN/../lib;${_CONDA_LIB}")
endif()

# ------------------ Find libusb-compat (usb.h / libusb) ------------------------
# --- Try the project's custom finder first (but don't fail if missing) ---
include(cmake/FindLibUSB.cmake OPTIONAL)

# Normalize variable names from various find scripts
# (some set LIBUSB_INCLUDE_DIR / LIBUSB_LIBRARY, others set *_DIRS / *_LIBRARIES)
if (LIBUSB_INCLUDE_DIR AND NOT LIBUSB_INCLUDE_DIRS)
  set(LIBUSB_INCLUDE_DIRS ${LIBUSB_INCLUDE_DIR})
endif()
if (LIBUSB_LIBRARY AND NOT LIBUSB_LIBRARIES)
  set(LIBUSB_LIBRARIES ${LIBUSB_LIBRARY})
endif()

# --- Fallback: locate libusb-compat (0.1 API: header is usb.h) directly ---
if (NOT LIBUSB_INCLUDE_DIRS OR NOT LIBUSB_LIBRARIES)
  # Header
  find_path(LIBUSB_INCLUDE_DIRS
    NAMES usb.h
    HINTS
      $ENV{CONDA_PREFIX}/include
      /usr/local/include
      /usr/include
  )
  # Library (lib name is usually 'usb' or 'usb-0.1')
  find_library(LIBUSB_LIBRARIES
    NAMES usb usb-0.1
    HINTS
      $ENV{CONDA_PREFIX}/lib
      /usr/local/lib
      /usr/lib
  )
endif()

# Hard fail if still not found
if (NOT LIBUSB_INCLUDE_DIRS OR NOT LIBUSB_LIBRARIES)
  message(FATAL_ERROR
    "libusb-compat (0.1) not found.\n"
    "Expected header 'usb.h' and library 'libusb[ -0.1].so' in your conda env.\n"
    "Hint: activate your env and/or install with: conda install -c conda-forge libusb-compat\n"
    "Or pass: -DLIBUSB_INCLUDE_DIRS=<path> -DLIBUSB_LIBRARIES=<path-to-libusb.so>"
  )
endif()

# Export to cache so subdirectories can use these variables too
set(LIBUSB_INCLUDE_DIRS ${LIBUSB_INCLUDE_DIRS} CACHE INTERNAL "libusb include dirs")
set(LIBUSB_LIBRARIES    ${LIBUSB_LIBRARIES}    CACHE INTERNAL "libusb libraries")

# --- Make headers visible to all targets (fixes your missing -I problem) ---
include_directories(
  ${PROJECT_SOURCE_DIR}/include
  ${LIBUSB_INCLUDE_DIRS}
)

# ------------------ Project layout ------------------
add_subdirectory(include)
add_subdirectory(lib)
add_subdirectory(src)
