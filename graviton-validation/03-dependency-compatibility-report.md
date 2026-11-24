# Dependency Compatibility Report — Applied Versions

| Dependency | Current Version | Finding | ARM64 Check Performed | Applied Version | Notes |
|---|---|---|---|---|---|
| cisd:jhdf5 | 19.04.1 | 01M87PAZSNPC84JHYAF1EBGXPJ | Listed the JAR's `native/jhdf5/`: no `aarch64-Linux` entry. Checked scijava metadata and sissource tags: 19.04.1 is the latest | 19.04.1 (unchanged) | No version with Linux/aarch64 natives exists. Added a `-Dnative.libpath.jhdf5` pass-through hook to the test task. Blocking until the user supplies an aarch64 libjhdf5.so (see 02-native-library-report.md) |

No other dependency is flagged MUST UPGRADE. No dependency versions were changed.

## Transitive Dependency Resolutions

| Transitive | Via | Resolution |
|---|---|---|
| cisd:base:18.09.0 (libnativedata, libunix; no aarch64-Linux) | cisd:jhdf5 | No change. NativeData has a built-in pure-Java fallback. TASSEL does not use the `unix` package |

## Build Environment Note (not ARM64 related)

`org.biojava.thirdparty:forester:1.039` depends on `openchart:openchart:1.4.2`, which cannot be resolved from Maven Central or maven.scijava.org.
This is a pre-existing problem and is unrelated to ARM64. The project build file was **not** changed for it.
For validation in this run, the dependency was excluded with a temporary Gradle init script outside the repository
(`allprojects { configurations.all { exclude group: 'openchart', module: 'openchart' } }`). TASSEL source has no reference to openchart.
