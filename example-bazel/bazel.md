# Install Bazel

[Install Doc](https://bazel.build/install/ubuntu?hl=en)

Download bazel installer from https://github.com/bazelbuild/bazel/releases
```bash
wget https://github.com/bazelbuild/bazel/releases/download/6.2.1/bazel-6.2.1-installer-linux-x86_64.sh
chmod +x bazel-version-installer-linux-x86_64.sh
./bazel-version-installer-linux-x86_64.sh --user
```

## Setup a minimum bazel project for testing

[Doc](https://github.com/bazelbuild/rules_go)

### Scaffold the go project
```bash
mkdir project_go
cd project_go
go mod init projectgo
```

### Create a simple go file and test file
```go
// main.go
func main() {
    fmt.Println("Hello World")
}

func Add(a, b int) int {
    return a + b
}

// main_test.go
func TestAdd(t *testing.T) {
    if Add(1, 2) != 3 {
        t.Error("1 + 2 did not equal 3")
    }
}
```

### Create bazel WORKSPACE file
```bash
touch WORKSPACE
cat WORKSPACE
```

```bazel
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
    name = "io_bazel_rules_go",
    sha256 = "6b65cb7917b4d1709f9410ffe00ecf3e160edf674b78c54a894471320862184f",
    urls = [
        "https://mirror.bazel.build/github.com/bazelbuild/rules_go/releases/download/v0.39.0/rules_go-v0.39.0.zip",
        "https://github.com/bazelbuild/rules_go/releases/download/v0.39.0/rules_go-v0.39.0.zip",
    ],
)

http_archive(
    name = "bazel_gazelle",
    sha256 = "727f3e4edd96ea20c29e8c2ca9e8d2af724d8c7778e7923a854b2c80952bc405",
    urls = [
        "https://mirror.bazel.build/github.com/bazelbuild/bazel-gazelle/releases/download/v0.30.0/bazel-gazelle-v0.30.0.tar.gz",
        "https://github.com/bazelbuild/bazel-gazelle/releases/download/v0.30.0/bazel-gazelle-v0.30.0.tar.gz",
    ],
)

load("@io_bazel_rules_go//go:deps.bzl", "go_register_toolchains", "go_rules_dependencies")
load("@bazel_gazelle//:deps.bzl", "gazelle_dependencies")

go_rules_dependencies()

go_register_toolchains(version = "1.20.5")

gazelle_dependencies()
```

### Create BUILD.bazel file
```bash
touch BUILD.bazel
cat BUILD.bazel
```

```bazel
load("@io_bazel_rules_go//go:def.bzl", "go_binary", "go_library", "go_test")
load("@bazel_gazelle//:def.bzl", "gazelle")

# gazelle:prefix projectgo
gazelle(name = "gazelle")
```

NOTE: use the mod name `projectgo` replace the prefix `# gazelle:prefix github.com/example/project` and get `# gazelle:prefix projectgo`

### Run gazelle to generate BUILD.bazel file
```bash
bazel run //:gazelle
```

Updated BUILD.bazel file
```bazel
load("@io_bazel_rules_go//go:def.bzl", "go_binary", "go_library", "go_test")
load("@bazel_gazelle//:def.bzl", "gazelle")

# gazelle:prefix projectgo
gazelle(name = "gazelle")

go_library(
    name = "projectgo_lib",
    srcs = ["main.go"],
    importpath = "projectgo",
    visibility = ["//visibility:private"],
)

go_binary(
    name = "projectgo",
    embed = [":projectgo_lib"],
    visibility = ["//visibility:public"],
)

go_test(
    name = "projectgo_test",
    srcs = ["main_test.go"],
    embed = [":projectgo_lib"],
)
```

## Set up java runtime toolchain for bazel test and bazel coverage

[rule_java](https://github.com/bazelbuild/rules_java)
[repositories](https://github.com/bazelbuild/rules_java/blob/master/java/repositories.bzl)
[jdk.WORKSPACE.tmpl](https://github.com/bazelbuild/bazel/blob/6.2.1/src/main/java/com/google/devtools/build/lib/bazel/rules/java/jdk.WORKSPACE.tmpl)

```
load("@bazel_tools//tools/jdk:remote_java_repository.bzl", "remote_java_repository")

remote_java_repository(
    name = "remotejdk20_linux",
    target_compatible_with = [
        "@platforms//os:linux",
        "@platforms//cpu:x86_64",
    ],
    sha256 = "0386418db7f23ae677d05045d30224094fc13423593ce9cd087d455069893bac",
    strip_prefix = "zulu20.28.85-ca-jdk20.0.0-linux_x64",
    urls = [
        "https://mirror.bazel.build/cdn.azul.com/zulu/bin/zulu20.28.85-ca-jdk20.0.0-linux_x64.tar.gz",
        "https://cdn.azul.com/zulu/bin/zulu20.28.85-ca-jdk20.0.0-linux_x64.tar.gz",
    ],
    version = "20",
)
```

NOTE: for some bazel version, the `target_compatible_with` is not available in `remote_java_repository`, use `exec_compatible_with` instead.

```
bazel coverage --tool_java_runtime_version=remotejdk_20 //...
```