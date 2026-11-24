# Code Scan Findings — Changes Applied

## Changes Applied

| File | Change | Finding |
|---|---|---|
| `src/main/java/net/maizegenetics/matrixalgebra/jni/net_maizegenetics_matrixalgebra_Matrix_BlasDoubleMatrix.c` | `#include <vecLib/cblas.h>` is now behind `#ifdef __APPLE__`. Other platforms (Linux x86_64/aarch64) use `#include <cblas.h>` | 01M87PB00DDRPNRGMH30PNRFC6 |
| `src/main/java/net/maizegenetics/matrixalgebra/Matrix/DoubleMatrixFactory.java` | No change. `System.loadLibrary("TasselBlas")` does not depend on architecture, and the existing `UnsatisfiedLinkError` catch already falls back to EJML | 01M87PB00DDRPNRGMH30PNRFC6 |
| `src/main/java/net/maizegenetics/matrixalgebra/Matrix/BlasDoubleMatrix.java` | No change. The JNI method declarations do not depend on architecture | 01M87PB00DDRPNRGMH30PNRFC6 |
| `build.gradle.kts` (test task) | Graviton JVM flags are added when `os.arch == aarch64`. The optional `jhdf5NativeLibPath` property or `JHDF5_NATIVE_LIBPATH` env var is passed as `-Dnative.libpath.jhdf5` | 01M87PB05BP614JK7RN0CX0GBY, 01M87PAZSNPC84JHYAF1EBGXPJ |

No `os.arch` checks exist in the source tree, so no architecture-detection code needed updating.
