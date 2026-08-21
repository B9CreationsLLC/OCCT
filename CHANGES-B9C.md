# B9Creations changes to Open CASCADE Technology

This is a fork of [Open CASCADE Technology](https://github.com/Open-Cascade-SAS/OCCT),
reduced to the subset B9Create requires: STEP import, shape healing, meshing, and STL export.

The fork exists to satisfy the source-availability requirement of the GNU LGPL v2.1, under
which OCCT is distributed. It is not intended as a general-purpose OCCT distribution.

Upstream base: tag `V8_0_1`.

## Changes

### 2026-08-21 — Reduce to the STEP -> heal -> mesh -> STL toolkit set

**Deletions only. No code was added or modified.**

Removed the `STEPCAFControl` package from `TKDESTEP` and the `DESTL` package from `TKDESTL`,
along with the toolkit-level `DEPLUGIN` registration units, the `STEPConstruct_Styles` /
`STEPConstruct_RenderingProperties` files that only `STEPCAFControl` used, and
`DESTEP_Provider` / `DESTEP_ConfigurationNode` from the `DESTEP` package. These are the
XDE/OCAF-aware STEP reader/writer and the DE provider registry; B9Create uses
`STEPControl_Reader` directly and does not use OCAF documents.

The rest of the `DESTEP` package is retained: `DESTEP_Parameters` is not part of the DE
provider surface but the parameter struct for the core STEP reader itself, referenced from
`STEPControl_Reader.hxx` and `StepData_StepModel.hxx`. It keeps a dependency on `TKDE` for
`DE_ShapeFixParameters`, which is why `TKDE` remains in the closure. `TKDE` itself depends
only on `TKernel`, `TKMath` and `TKBRep`.

Their `#include <XCAFPrs_*>` / `<XCAFDoc_*>` directives were the only thing linking
`TKDESTEP` and `TKDESTL` to `TKXCAF`, and through it to `TKVCAF` -> `TKV3d` -> `TKService`.
Removing them drops the Visualization and ApplicationFramework modules from the dependency
closure entirely, reducing it from 24 toolkits to 14 and eliminating every third-party
runtime dependency (FreeType, FreeImage, FFmpeg, TBB, OpenVR, Tcl/Tk, jemalloc, zlib).

Resulting closure:

    FoundationClasses    TKernel TKMath
    ModelingData         TKG2d TKG3d TKGeomBase TKBRep
    ModelingAlgorithms   TKGeomAlgo TKTopAlgo TKShHealing TKMesh
    DataExchange         TKDE TKXSBase TKDESTEP TKDESTL

No longer built: TKPrim, TKBO, TKHLR, TKService, TKV3d, TKCDF, TKLCAF, TKCAF, TKVCAF,
TKXCAF, and everything in the Visualization and Draw modules.

Also removed, as they do not affect the build: `tests/`, `data/`, `dox/`, `samples/`,
`resources/` (resource data is compiled into the libraries as generated `.pxx` byte arrays;
`resources/` is only needed when `BUILD_RESOURCES=ON` regenerates them), `adm/vcpkg/`, and
the per-toolkit `GTests/` directories.

CMake lists touched: 8. All changes are deletions.

### Cherry-picked from upstream master

- `7d2efad9c8` (`6e6b820e94` after pick) — "Mesh - Fix iterator invalidation in Delaun triangle-kill cascade (#1450)",
  a fix in `BRepMesh_Delaun.cxx` landed after `V8_0_1`. It is on B9Create's meshing path, so
  it is applied here rather than lost by pinning to the tag. Authored by Open Cascade
  contributors; authorship preserved.

### Also removed: upstream CI

`.github/workflows/` and `.github/actions/` drove OCCT's own multiplatform build, doc
generation and DRAW test suite against the `tests/`, `dox/` and `data/` trees removed above.
Left in place they would produce a failing Actions run on every push to this fork. Removing
them is equivalent to disabling Actions in the repository settings; either is fine, but
deleting them keeps the repository self-consistent.

### Fix: remove `adm/RESOURCES`

Removing `resources/` alone was not sufficient. `adm/RESOURCES` lists the resource
directories that the unguarded install loop at `CMakeLists.txt:1318-1327` turns into
`install()` rules, and `OCCT_INSTALL_FILE_OR_DIR()` performs no existence check - so the
install step failed with "file INSTALL cannot find .../resources/DrawResources".

`FILE_TO_LIST()` already guards on `EXISTS`, so deleting `adm/RESOURCES` yields an empty
list and the loop emits no rules. The other install rules that reference removed trees are
already guarded and need no change: `tests/` by `INSTALL_TEST_CASES`, `data/` by `DRAWEXE`
being in `BUILD_TOOLKITS`, and `dox/resources/overview.html` by the documentation block.

No resource files are needed at runtime for the STEP -> heal -> mesh -> STL path; the message
and units data are compiled into the libraries as generated `.pxx` byte arrays.

### Also removed: `src/Deprecated/`

972 backward-compatibility alias headers (4.3 MB) in `src/Deprecated/NCollectionAliases/`,
plus the unconditional `install(DIRECTORY ...)` rule that shipped them.

They belong to no toolkit and are compiled by nothing. Verified: no `#include` of any of the
972 exists anywhere in OCCT's own sources or in B9Create. The single apparent hit,
`Poly_HArray1OfTriangle`, is the string literal `"PPoly_HArray1OfTriangle"` in `TKStd`'s
persistence type map - not an include, and `TKStd` is not in the closure.

## Building

The shipped binaries were built on Windows x64 with:

- Visual Studio 2019 Community 16.11.53, MSVC toolset 14.29.30133 (`cl` 19.29.30159)
- CMake 3.20.21032501-MSVC_2 and Ninja 1.10.2, as bundled with Visual Studio 2019

The build options are in `CMakePresets.json`. From an x64 Native Tools Command Prompt for
VS 2019:

    cmake --preset x64-release
    cmake --build --preset x64-release
    cmake --install out/build/x64-release

The distributed package is the `bin/`, `include/` and `lib/` directories of
`out/install/x64-release`. B9Create itself builds with Visual Studio 2022; the v142
binaries are compatible with the v143 toolset.
