# ARM64 Compatibility Validation — Summary

## Project Overview
TASSEL 5 renovation (`net.maizegenetics:5.2.96`) is a Gradle (Kotlin DSL) Java/Kotlin project with a JDK 21 toolchain.
It is deployed host-based: there is no Dockerfile and no start script. ARM64-relevant native code:
- the project-owned JNI `libTasselBlas` (optional, with an EJML fallback)
- `cisd:jhdf5:19.04.1` `libjhdf5` (required for HDF5 I/O)

The validation host was x86_64. No ARM64 runtime was available.

## Exit Criteria Checklist
| # | Criterion | Status | Notes |
|---|-----------|--------|-------|
| 1 | Compiles on ARM64 without errors | ⚠️ PARTIAL | Pure JVM bytecode compiles (`build -x test` PASS on x86_64). JNI C cross-compiles for aarch64. Not run on an ARM64 host |
| 2 | Native libraries load on ARM64 | ❌ FAIL (blocked) | libTasselBlas: EJML fallback works. libjhdf5: no aarch64-Linux binary exists in any jhdf5 release, so the user must build and supply it |
| 3 | Blocking dependency updates applied | ⚠️ N/A | No ARM64-capable jhdf5 version exists to upgrade to (19.04.1 is the latest). A `native.libpath.jhdf5` hook was added |
| 4 | No ARM64-specific runtime errors | ⚠️ NOT VERIFIED | No ARM64 host. HDF5 is expected to raise `UnsatisfiedLinkError` until libjhdf5 is supplied |
| 5 | Container image builds on ARM64 | N/A | Not containerized |
| 6 | Test suite executes on ARM64 | ⚠️ NOT VERIFIED | Executed on x86_64: 19 run, 16 failed. Identical to baseline |
| 7 | No ARM64-related test failures | ✅ PASS (x86_64) | All failures are INFRA/PRE-EXISTING (missing `dataFiles/`) |
| 8 | Application starts on ARM64 | ⚠️ NOT VERIFIED | x86_64 smoke test with Graviton flags passes |
| 9 | JVM flags accepted without errors | ✅ PASS | Corretto 21 accepted all flags |

## Findings Addressed
| Finding ID | Title | Status |
|---|---|---|
| 01M87PB00DDRPNRGMH30PNRFC6 | TasselBlas JNI C source depends on macOS-only vecLib header | Resolved in source. Linux uses `<cblas.h>`, and the aarch64 cross-compile is verified. Building and shipping libTasselBlas.so needs OpenBLAS on a Graviton host and is optional, because the EJML fallback is in place |
| 01M87PAZSNPC84JHYAF1EBGXPJ | cisd:jhdf5 19.04.1 Linux/aarch64 native libjhdf5 unverified | Verified **missing** (only 32-bit `arm-Linux`). No newer release exists. Version kept. A `-Pjhdf5NativeLibPath` / `JHDF5_NATIVE_LIBPATH` hook was added. **NOT COMPLETE:** the user must build an aarch64 libjhdf5.so |
| 01M87PB05BP614JK7RN0CX0GBY | No Graviton-optimized runtime JVM flags defined | Resolved. JDK 21 Graviton flags are added to the test JVM on aarch64 and validated. Launch-script guidance is in 05-jvm-configuration.md |

## Changes Made
- `src/main/java/net/maizegenetics/matrixalgebra/jni/net_maizegenetics_matrixalgebra_Matrix_BlasDoubleMatrix.c`: platform-conditional CBLAS include (see 02, 04).
- `build.gradle.kts` test task: aarch64-only Graviton JVM flags and the optional `native.libpath.jhdf5` pass-through (see 03, 04, 05).
- No dependency versions, Java version or JDK distribution were changed.

## Remaining Concerns
1. **libjhdf5 for Linux/aarch64 (blocking for HDF5 features).** Build JHDF5 19.04.1 JNI (`https://sissource.ethz.ch/sispub/jhdf5`) against HDF5 1.10.x on a Graviton host.
   Then run with `-Dnative.libpath.jhdf5=/path/libjhdf5.so`, or `./gradlew test -Pjhdf5NativeLibPath=/path/libjhdf5.so` for tests.
2. **Optional:** build libTasselBlas.so against OpenBLAS on Graviton for native BLAS speed (command in 02).
3. **Pre-existing, not ARM64:** `openchart:openchart:1.4.2` (via forester 1.039) is unresolvable, and `dataFiles/` test data is missing from the repository.
4. Run the build, tests and startup on an actual Graviton instance. Performance benchmarking is the user's responsibility.
