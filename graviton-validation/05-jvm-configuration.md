# Graviton JVM Configuration

## Target Java Version
JDK 21 (Gradle toolchain `languageVersion = 21`, bytecode target 21). Validated with Amazon Corretto 21.0.13.1.

## Applied JVM Flags
The flags are applied in the only existing JVM-flag location, the Gradle `test` task `jvmArgs` (`build.gradle.kts`).
They are added only when the build runs on `aarch64`, and the existing `-Xmx10g` is preserved.

| Flag | Applicable JDK Versions | Workload Type | Expected Impact |
|------|------------------------|---------------|-----------------|
| -XX:-TieredCompilation | 11, 17, 21+ | Moderate lock contention / long-running compute | Less compilation overhead; C2-only code |
| -XX:ReservedCodeCacheSize=64M | 11, 17, 21+ | General | Code cache sized for C2-only compilation |
| -XX:InitialCodeCacheSize=64M | 11, 17, 21+ | General | Avoids code cache growth at runtime |
| -Xss1m | 11, 17, 21+ | Multi-threaded (TASSEL plugins use thread pools) | Lower per-thread memory |

TASSEL has no start script, Dockerfile or service unit in this repository. Wherever TASSEL is launched on Graviton
(for example the `run_pipeline.pl`/`start_tassel.pl` wrappers distributed separately), add the same flags. If HDF5 is needed,
also add `-Dnative.libpath.jhdf5=/path/to/aarch64/libjhdf5.so`.

## Flags NOT Applied (Version-Gated)
| Flag | Reason Not Applied |
|------|--------------------|
| -XX:CompilationMode=high-only | Not recommended for JDK 21+ (removed in some builds) |
| -XX:CICompilerCount=2 | "Test first" for JDK 21+. It is not needed for this batch workload |
| -XX:+UnlockDiagnosticVMOptions -XX:+UseAESCTRIntrinsics | On by default in JDK 21+. No cryptographic workload |

## Flag Validation
```
$ /usr/lib/jvm/java-21-amazon-corretto.x86_64/bin/java -Xmx10g -XX:-TieredCompilation -XX:ReservedCodeCacheSize=64M -XX:InitialCodeCacheSize=64M -Xss1m -version
openjdk version "21.0.13.1" 2026-11-17 LTS
OpenJDK Runtime Environment Corretto-21.0.13.1.2 (build 21.0.13.1+2-LTS)
OpenJDK 64-Bit Server VM Corretto-21.0.13.1.2 (build 21.0.13.1+2-LTS, mixed mode, sharing)
rc=0
```
All flags were accepted. They were validated on x86_64 because no ARM64 host was available. These are standard HotSpot flags that are also accepted on aarch64.

## Configuration Profiles
There is a single profile, applied automatically when `os.arch == aarch64`.
