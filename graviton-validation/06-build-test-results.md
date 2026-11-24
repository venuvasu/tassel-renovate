# Build & Test Validation Results

## Build Environment
- **Architecture:** `x86_64`. No ARM64 host, qemu or Docker was available, so ARM64 execution could not be validated.
- **Java Runtime:** Amazon Corretto 21.0.13.1 (`JAVA_HOME=/usr/lib/jvm/java-21-amazon-corretto.x86_64`). This matches the project toolchain 21. The default shell `java` (Corretto 25) was not changed.
- **Build Tool:** Gradle 8.13 (wrapper)
- **Build Environment Notes:** `JAVA_HOME` was set for the session only. A temporary init script at `/tmp/exclude-openchart.gradle`, outside the repo,
  excluded the unresolvable transitive `openchart:openchart:1.4.2` (pre-existing, not ARM64 related).
- **Cross-compile tool:** `zig cc` 0.17.1 (PyPI `ziglang`, in a temporary venv) was used to verify that the JNI C source compiles for aarch64-linux-gnu.

## Build Attempts
| # | Command | Result | Notes |
|---|---------|--------|-------|
| 1 | `./gradlew clean build` | ❌ FAIL | `Could not find openchart:openchart:1.4.2` (required by forester:1.039). Pre-existing, unrelated to ARM64 |
| 2 | `./gradlew clean build -I /tmp/exclude-openchart.gradle` | ❌ FAIL | Compilation passed. `:test`: 19 tests completed, 16 failed, 1 skipped, then `Gradle Test Executor 1 finished with non-zero exit value 1` |
| 3 | Baseline (original code, `git stash`) `./gradlew clean test -I /tmp/exclude-openchart.gradle` | ❌ FAIL | Identical result to attempt 2 (19/16/1, executor exit 1), so the failures are pre-existing |
| 4 | `./gradlew clean build -x test -I /tmp/exclude-openchart.gradle` | ✅ PASS | Kotlin and Java compile, jar `sTASSEL.jar`, copyDependencies |
| 5 | `zig cc -target aarch64-linux-gnu -O3 -mcpu=neoverse_n1 -fPIC -shared ... BlasDoubleMatrix.c` | ✅ PASS | `ELF 64-bit LSB shared object, ARM aarch64`. The original source fails with `'vecLib/cblas.h' file not found` |

## Test Failure Classification
| Test | Class | Failure Type | Root Cause | Blocking |
|------|-------|-------------|------------|----------|
| testUnreplicatedTaxa, testHeterozygotes, ... | DiscreteSitesTest, EqtlAssociationPluginTest, GenosToABHPluginTest, ... | INFRA / PRE-EXISTING | `dataFiles/...` test data is not in the repository (`No such file or directory`) | No |
| testMigrateT4WithGenotypes | MigrateHDF5FromT4T5Test | INFRA / PRE-EXISTING | `dataFiles/CandidateTests/regHmpInT4.hmp.h5` is missing | No |
| testPlinkLoadPlugin (skipped) | PlinkLoadPluginTest | INFRA | Missing `.ped` test file | No |
| Test executor exit 1 | (test JVM) | PRE-EXISTING | Same on baseline. A test terminates the JVM after missing data | No |

No failures were classified as ARM64. On a real Graviton host, every HDF5-backed test is expected to fail with `UnsatisfiedLinkError`
until an aarch64 `libjhdf5.so` is supplied through `-Pjhdf5NativeLibPath=...` (finding 01M87PAZSNPC84JHYAF1EBGXPJ, blocking).

## Final Build (Scoring Basis)
- **Command:** `./gradlew clean build -x test -I /tmp/exclude-openchart.gradle`
- **Result:** ✅ PASS (on x86_64)
- **Rationale:** All test failures are INFRA or PRE-EXISTING, and the baseline reproduces them identically. Compilation is the scoring basis. ARM64 execution is still pending on a Graviton host.

## Container Validation (if applicable)
N/A. The repository has no Dockerfile (host-based deployment).

## Startup Validation
- **Smoke test:** Ran `DoubleMatrixFactory` initialization and a matrix multiply with the Graviton flags on JDK 21:
  `os.arch=amd64 factory=ejml`, `mult ok: 7.0`. This exercises the libTasselBlas-absent fallback path, which Graviton will use unless libTasselBlas.so is built.
- **Application starts without errors:** ✅ Yes (on x86_64). Not verified on ARM64 because no host was available.
- **JVM flags accepted:** ✅ Yes (`java <flags> -version` rc=0)
- **Runtime crashes:** None observed. HDF5 features are expected to fail on aarch64 until libjhdf5 is supplied.
