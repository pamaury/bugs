This project triggers to bug in rules_rust:
- rules_rust does not correctly sets the features of a crate and this causes
  serde_with to not compile
- rules_rust creates a dependency on a optional dependency that is not used

# Mishandling of no_default_features

The first issue that can be observed by running
```bash
bazelisk build @crate_index//:serde_with
```
This does not compile because `hashbrown::raw` is private. This is because
the `raw` feature of `hashbrown` is not enabled in [BUILD.hashbrown-0.12.3.baze](crates/BUILD.hashbrown-0.12.3.bazel)
despite that feature being specified by [indexmap 1.9.3](https://github.com/bluss/indexmap/blob/861fad73de14b7f08d2dc56ed83607aef621b42e/Cargo.toml)
on which `serde_with` depends. Probably this is because `indexmap` also specifies no default:
```toml
[dependencies.hashbrown]
version = "0.12"
default-features = false
features = ["raw"]
```
The problem can be fixed by manually adding a `crate.annotation` to add an extra feature to
the crate:
```python
    annotations = {
        # This is a workaround: presumably because of a rules_rust bug, indexmap 1.9.3 depends
        # on hashbrown 0.12 with no defaults and the `raw` feature but rules_rust does not add this
        # feature to crate which prevents the crate from compiling.
        "hashbrown": [crate.annotation(
            crate_features = ["raw"],
        )],
        # Same problem with `deranged` (pulled by `time`).
        "deranged": [crate.annotation(
            crate_features = ["powerfmt"],
        )],
    },
```
Note that the same problem occurs with `deranged` which is pulled by `time`.

This root cause of the issue seems to be the use of `default-features = false` in `Cargo.toml`.

# Optional and unused dependencies that are renamed are pulled

The second issue is not critical but can cause rules_rust to pull many unneeded dependencies.
In the repository, notice that we are using the default features of `serde_with` which do not
enabled `indexmap_1` by default. `indexmap 1.9.3` is marked as optional in [`serde_with`](https://github.com/jonasbb/serde_with/blob/master/serde_with/Cargo.toml):
```toml
indexmap_1 = {package = "indexmap", version = "1.8", optional = true, default-features = false, features = ["serde-1"]}
```
Therefore rules_rust should not pull `indexmap 1.9.3`. But it seems that it does because `serde_with` is
renaming the crate. This translates to the following in [BUILD.serde_with-3.4.0.bazel](crates/BUILD.serde_with-3.4.0.bazel):
```python
rust_library(
    name = "serde_with",
    srcs = glob(["**/*.rs"]),
    aliases = {
        "@crate_index__indexmap-1.9.3//:indexmap": "indexmap_1",
    },
```
This is the only way that `serde_with` "depends" on `indexmap`.
