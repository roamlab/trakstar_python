This repo implements a Python wrapper around the [ATC3DGTracker](https://github.com/ChristophJud/ATC3DGTracker) driver package.

## Environment Setup

Turn on trakSTAR and connect trakSTAR via USB.

Create a conda/mamba environment:

```bash
mamba create -n trakstar_python
mamba activate trakstar_python
mamba install libusb-compat numpy
```

Clone this repo:

```bash
git@github.com:roamlab/trakstar_python.git
```

Build:

```bash
# On repo's root directory:
rm -rf build && mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH="$CONDA_PREFIX" ..
make -j
```

Add a udev permission file:

```bash
sudoedit /etc/udev/rules.d/99-trakstar.rules
# paste the following line in
SUBSYSTEM=="usb", ATTRS{idVendor}=="04b4", ATTRS{idProduct}=="1005", GROUP="plugdev", MODE="0666"
```

After adding the file, with trakSTAR plugged in and powered on:

```bash
sudo udevadm control --reload
sudo udevadm trigger
# Unplug trakSTAR USB then replug if you want to be extra careful...
```

Run test script:
```bash
./src/atcTest
```



## Bug fixes

### How to properly install libusb?
The ATC3DGTracker codebase was built over 10 years ago. `libusb-0.1` was the library back then. Files like `lib/PointATC3DG.cpp` relies on the header `usb.h` to build. `lib/PointATC3DG.cpp` also contains APIs calls to libusb in the format of `usb_init`, `usb_open`, `usb_close`, etc., which were the standard naming convention for `libusb-0.1`. Both the `usb.h` header and the API function naming convention were changed in `libusb-1.0`. The new header is `libusb.h`, and the typical API functions are in the format of `libusb_init`, `libusb_open`, etc. Directly using `apt` to install `libusb-0.1-dev` might seem like the go-to option, but some Linux distros no longer ship `libusb-0.1-dev`, and installing an old libusb package via `apt` (a system-wide install) risks polluting the environment for other projects. If we don't want to modify the source code of ATC3DGTracker, the most convenient method is to install `libusb-compat` in conda/mamba. `libusb-compat` is a compatibility layer/wrappepr that implements the 0.1 API on top of `libusb-1.0`. When installing via conda, the dependency check also installed the `libusb-1.0` runtime in the same conda environment. It also provides the `usb.h` header and allows old code that were built with `libusb-0.1` to continue call the API functions in the `usb_*` convention.

### CMake error - “usb.h: No such file or directory”
`lib/PointATC3DG.cpp` requires `usb.h`, as mentioned above. However, we did not do an apt install. We installed via conda, which means `usb.h` is placed under `/home/miniforge3/...`. The `CMakeLists.txt` in the original `ATC3DGTracker` implementation expects an `apt` installed `libusb`, so the compiler was never told to search the conda include path `$CONDA_PREFIX/include`. To resolve this, we need to modify the top-level `CMakeLists.txt` to use the custom finder `FindLibUSB.cmake` if present, or fall back to `find_path/find_library` to locate `usb.h` and `libusb` in `$CONDA_PREFIX`. We also call `include_directories(${LIBUSB_INCLUDE_DIRS})` globally so all targets get `-I$CONDA_PREFIX/include`. With these changes, every compile command includes the conda headers, so `<usb.h>` is resolvable. We also need to add global RPATH in the top-level `CMakeLists.txt`, because at runtime dynamic loader couldn’t find some shared libraries such as `libatclib.so.*`. This is done by setting `CMAKE_INSTALL_RPATH` to `"$ORIGIN/../lib;${CONDA_PREFIX}/lib"` so binaries find project libs next to them and also fall back to the conda env.

### Why changing the CMake command?
We are adding `-DCMAKE_PREFIX_PATH="$CONDA_PREFIX"` to tell any “find” logic to look inside our conda env first. While we have a fallback in the updated `CMakeLists.txt`, this is more expllicit. We also don't need `-DCMAKE_INSTALL_PREFIX=/usr/local`, as it only sets where `make install` puts files and we don't need `make install`. `make install` copies build artifacts to the install prefix (e.g., `/usr/local`) in a nice layout, but we have just added a global RPATH in the CMake, so libraries in `build/lib` and `libusb` in `$CONDA_PREFIX/lib` are discoverable. We can run any binaries directly from the build tree.

### Why modify include/PointATC3DG.h?
The modification to `include/PointATC3DG.h` are: 
- `#define BIRD_VENDOR 0x04b4`
- `#define BIRD_PRODUCT 0x1005`

These are needed because the original ATC3DGTracker codebase had incorrect IDs that prevented us from finding/opening the device. Each hardware manufacturer gets assigned a Vendor ID (VID) by the USB-IF (USB Implementation Forum). Logitech gets `046D`, in our case we see "Cypress Semiconductor Corp." with VID `04b4`. Each product also gets a Product ID (PID) chosen by that manufacturer to distinguish one product/model from another. OS uses these information to choose the right driver and ensure we can communicate with the device. VID and PID can be found by running `lsusb` with the device connected, or running `dmesg -w` with the device unplugged, then plugged the device in.  

