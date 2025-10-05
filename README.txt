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

### How to properly install libusb
The ATC3DGTracker codebase was built over 10 years ago. `libusb-0.1` was the library back then. Files like `lib/PointATC3DG.cpp` relies on the header `usb.h` to build. `lib/PointATC3DG.cpp` also contains APIs calls to libusb in the format of `usb_init`, `usb_open`, `usb_close`, etc., which were the standard naming convention for `libusb-0.1`. Both the `usb.h` header and the API function naming convention were changed in `libusb-1.0`. The new header is `libusb.h`, and the typical API functions are in the format of `libusb_init`, `libusb_open`, etc. Directly using `apt` to install `libusb-0.1-dev` might seem like the go-to option, but some Linux distros no longer ship `libusb-0.1-dev`, and installing an old libusb package via `apt` (a system-wide install) risks polluting the environment for other projects. If we don't want to modify the source code of ATC3DGTracker, the most convenient method is to install `libusb-compat` in conda/mamba. `libusb-compat` is a compatibility layer/wrappepr that implements the 0.1 API on top of libusb-1.0. When installing via conda, the dependency check also installed the libusb-1.0 runtime in the same conda environment. It also provides the `usb.h` header and allows old code that were built with `libusb-0.1` to continue call the API functions in the `usb_*` convention.

### CMAKE error “usb.h: No such file or directory”:
`lib/PointATC3DG.cpp` requires 

