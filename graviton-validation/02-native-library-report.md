# Native Library Report — Resolution Details

## Resolution Details

### 1. TasselBlas JNI (project-owned) — finding 01M87PB00DDRPNRGMH30PNRFC6

- **Library:** `libTasselBlas` (`System.loadLibrary("TasselBlas")` in `DoubleMatrixFactory.java`; native methods in `BlasDoubleMatrix.java`)
- **Problem:** `src/main/java/net/maizegenetics/matrixalgebra/jni/net_maizegenetics_matrixalgebra_Matrix_BlasDoubleMatrix.c`
  included `<vecLib/cblas.h>`, the macOS Accelerate header, so it could not compile on Linux (x86_64 or aarch64).
- **Resolution:** The include is now platform-conditional:
  ```c
  #ifdef __APPLE__
  #include <vecLib/cblas.h>
  #else
  #include <cblas.h>
  #endif
  ```
  On macOS the code path is unchanged. Linux now uses the standard CBLAS header from OpenBLAS or netlib CBLAS.
- **Verification (this run, x86_64 host, cross-compile):**
  - The original source fails for `aarch64-linux-gnu` with `fatal error: 'vecLib/cblas.h' file not found`.
  - The updated source compiles with `zig cc -target aarch64-linux-gnu -O3 -mcpu=neoverse_n1 -fPIC -shared` against the reference-LAPACK v3.12.0 CBLAS headers and JDK 21 JNI headers.
    `file` reports: `ELF 64-bit LSB shared object, ARM aarch64`.
  - This build was only a compilation check. It was not linked against an aarch64 OpenBLAS, so it is **not** committed.
    The project ships no prebuilt `libTasselBlas` for any platform.
- **Building the library on a Graviton host (user action, optional):**
  ```bash
  sudo dnf install -y openblas-devel lapack-devel gcc   # Amazon Linux 2023 package names
  cd src/main/java/net/maizegenetics/matrixalgebra/jni
  gcc -shared -fPIC -O3 -mcpu=neoverse-n1 \
      -I"$JAVA_HOME/include" -I"$JAVA_HOME/include/linux" \
      net_maizegenetics_matrixalgebra_Matrix_BlasDoubleMatrix.c \
      -o libTasselBlas.so -lopenblas -llapack
  file libTasselBlas.so   # must show "ARM aarch64"
  # run TASSEL with -Djava.library.path=<dir containing libTasselBlas.so>
  ```

### 2. cisd:jhdf5 19.04.1 `libjhdf5` — finding 01M87PAZSNPC84JHYAF1EBGXPJ

- **Check performed:** Downloaded `cisd:jhdf5:19.04.1` from `maven.scijava.org` and listed its `native/jhdf5/` entries:

  | Directory | Binary | Usable on Linux/Graviton |
  |---|---|---|
  | `arm-Linux/libjhdf5.so` | ELF 32-bit ARM EABI5 | No (32-bit ARM, not aarch64) |
  | `amd64-Linux/libjhdf5.so` | x86_64 | No |
  | `amd64-Windows/jhdf5.dll` | x86_64 | No |
  | `x86_64-Mac OS X/libjhdf5.jnilib` | x86_64 | No |
  | `aarch64-Mac OS X/libjhdf5.jnilib` | Apple M1 | No (macOS only) |
  | `aarch64-Linux/` | **absent** | — |

- **Upgrade availability:** 19.04.1 is the latest version in the scijava `maven-metadata.xml`
  (12.02.3 … 19.04.1) and the latest tag in `sissource.ethz.ch/sispub/jhdf5`.
  Maven Central has no `cisd:jhdf5`. No version ships Linux/aarch64 natives, so the version is kept.
- **Transitive `cisd:base:18.09.0`:** `libnativedata` and `libunix` also have no aarch64-Linux build.
  `NativeData` falls back to pure Java when its native library fails to load (`useNativeLib=false`, or force it with `-Dnativedata.javamode=true`).
  TASSEL does not use `ch.systemsx.cisd.base.unix`.
- **Resolution in this run:** Added a hook to the `test` task in `build.gradle.kts`. The Gradle property `-Pjhdf5NativeLibPath=<file>` or the
  environment variable `JHDF5_NATIVE_LIBPATH` is passed through as `-Dnative.libpath.jhdf5=<file>`.
  `ch.systemsx.cisd.base.utilities.NativeLibraryUtilities` uses this property to load a user-supplied library file.
  For production runs, pass the same `-Dnative.libpath.jhdf5=/path/to/libjhdf5.so` to the TASSEL JVM.
- **Status: BLOCKING (user action required).** HDF5 I/O has no pure-Java fallback. Until a Linux/aarch64 `libjhdf5.so` is built from the JHDF5 19.04.1 source
  (`https://sissource.ethz.ch/sispub/jhdf5`, `source/c`, built against HDF5 1.10.x on an aarch64 host) and supplied as shown above,
  the HDF5 genotype, TOPM and TBT features (38 source files under `ch.systemsx.cisd.hdf5.*`) will throw `UnsatisfiedLinkError` on Graviton.
  This run could not build it: the host is x86_64, and HDF5's configure step must execute target binaries. No aarch64 runtime, qemu or Docker was available.

## Pure Java Fallbacks

| Native library | Fallback | Status |
|---|---|---|
| libTasselBlas | EJML (`DoubleMatrixFactory` catches `UnsatisfiedLinkError` and selects `FactoryType.ejml`) | Already present, no code change. Smoke-tested in this run: `factory=ejml`, matrix multiply correct |
| cisd:base libnativedata | Built-in Java implementation in `NativeData` | Automatic |
| libjhdf5 | None available | Blocking: user must supply an aarch64 `libjhdf5.so` |
