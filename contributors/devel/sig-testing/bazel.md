# Build and test with Bazel

Building and testing Kubernetes with  Bazel is supported but not yet default.

Bazel is used to run all Kubernetes PRs on [Prow](https://prow.k8s.io),
as remote caching enables significantly reduced build and test times.

Some repositories (such as kubernetes/test-infra) have switched to using Bazel
exclusively for all build, test, and release workflows.

Go rules are managed by the [`gazelle`](https://github.com/bazelbuild/rules_go/tree/master/go/tools/gazelle)
tool, with some additional rules managed by the [`kazel`](https://git.k8s.io/repo-infra/kazel) tool.
These tools are called via the `hack/update-bazel.sh` script.

Instructions for installing Bazel
can be found [here](https://www.bazel.io/versions/master/docs/install.html).
Please note that until [this Bazel
issue](https://github.com/bazelbuild/rules_docker/issues/454) is fixed,
`/usr/bin/env python` must be python2 in order for all the Bazel commands listed
below to succeed.

Several convenience `make` rules have been created for common operations:

* `make bazel-build`: builds all binaries in tree (`bazel build -- //...
  -//vendor/...`)
* `make bazel-test`: runs all unit tests (`bazel test --config=unit -- //...
  //hack:verify-all -//build/... -//vendor/...`)
* `make bazel-test-integration`: runs all integration tests (`bazel test
  --config integration //test/integration/...`)
* `make bazel-release`: builds release tarballs, Docker images (for server
  components), and Debian images (`bazel build //build/release-tars`)

You can also interact with Bazel directly; for example, to run all `kubectl` unit
tests, run

```console
$ bazel test //pkg/kubectl/...
```

## Planter
If you don't want to install Bazel, you can instead try using the unofficial
[Planter](https://git.k8s.io/test-infra/planter) tool,
which runs Bazel inside a Docker container.

For example, you can run
```console
$ ../test-infra/planter/planter.sh make bazel-test
$ ../test-infra/planter/planter.sh bazel build //cmd/kubectl
```

## Cross-compilation
Cross-compilation across OSes and architectures is now supported, though it
hasn't been extensively tested, so some issues may remain.

To enable cross-compilation, use one of the config settings defined in the root
`.bazelrc` in kubernetes/kubernetes; for example, to cross-compile `kubectl` to
linux/arm, you can run
```
bazel build --config=cross:linux_arm //cmd/kubectl
```

Note that some targets use CGO, and require a crossbuild toolchain (including
a C compiler, linker, and target libraries). In debian-like systems, these
are usually installable through package groups like `crossbuild-essential-armhf`.

If you don't want to install the crossbuild toolchain, you can use the
crossbuild version of planter by setting `CROSS=y` in your environment.

Known issues:
* The Bazel `CROSSTOOL` is currently configured primarily
  for the debian-like crossbuild toolchains described above. Potential future
  work includes better autodetection of the crossbuild toolchains, or even
  automatically downloading necessary crossbuild toolchain dependencies.
* The "convenience" make targets for Bazel have not yet been updated to
  support cross-compilation.
* Currently only one platform is configurable in a single Bazel run, so building
  the full suite of platforms requires multiple Bazel calls. Artifacts must
  additionally be copied out of the Bazel output tree after each call.
* Because only one platform is configurable at a time, building multi-arch
  manifest lists of container images is not supported.

## Continuous Integration

There are several bazel CI jobs:
* [ci-kubernetes-bazel-build](http://k8s-testgrid.appspot.com/google-unit#bazel-build): builds everything
  with Bazel
* [ci-kubernetes-bazel-test](http://k8s-testgrid.appspot.com/google-unit#bazel-test): runs unit tests in
  with Bazel

Similar jobs are run on all PRs; additionally, several of the e2e jobs use
Bazel-built binaries when launching and testing Kubernetes clusters.

## Updating `BUILD` files

To update `BUILD` files, run:

```console
$ ./hack/update-bazel.sh
```

To prevent Go rules from being updated, consult the [gazelle
documentation](https://github.com/bazelbuild/rules_go/tree/master/go/tools/gazelle).

Note that much like Go files and `gofmt`, `BUILD` files have standardized,
opinionated style rules, and running `hack/update-bazel.sh` will format them for you.

If you want to auto-format `BUILD` files in your editor, use of
[Buildifier](https://github.com/bazelbuild/buildtools/blob/master/buildifier/README.md)
is recommended.

Updating the `BUILD` file for a package will be required when:
* Files are added to or removed from a package
* Import dependencies change for a package
* A `BUILD` file has been updated and needs to be reformatted
* A new `BUILD` file has been added (parent `BUILD` files will be updated)

## Known issues and limitations

### The CC toolchain is not fully hermetic
Bazel requires several tools and development packages to be installed in the system, including `gcc`, `g++`, `glibc and libstdc++ development headers` and `glibc static development libraries`. Please check your distribution for exact names of the packages. Examples for some commonly used distributions are below:

|      Dependency       | Debian/Ubuntu                 | CentOS                         | OpenSuSE                                |
| :-------------------: | ----------------------------- | ------------------------------ | --------------------------------------- |
|   Build essentials    | `apt install build-essential` | `yum groupinstall development` | `zypper install -t pattern devel_C_C++` |
|        GCC C++        | `apt install g++`             | `yum install gcc-c++`          | `zypper install gcc-c++`                |
| GNU Libc static files | `apt install libc6-dev`       | `yum install glibc-static`     | `zypper install glibc-devel-static`     |

If any of these packages change, they may also cause spurious build failures
as described in [this issue](https://github.com/bazelbuild/bazel/issues/4907).

An example error might look something like
```
ERROR: undeclared inclusion(s) in rule '//vendor/golang.org/x/text/cases:go_default_library.cgo_c_lib':
this rule is missing dependency declarations for the following files included by 'vendor/golang.org/x/text/cases/linux_amd64_stripped/go_default_library.cgo_codegen~/_cgo_export.c':
  '/usr/lib/gcc/x86_64-linux-gnu/7/include/stddef.h'
```

The only way to recover from this error is to force Bazel to regenerate its
automatically-generated CC toolchain configuration by running `bazel clean
--expunge`.

Improving cgo cross-compilation may help with all of this.

### Changes to Go imports requires updating BUILD files
The Go rules in `BUILD` and `BUILD.bazel` files must be updated any time files
are added or removed or Go imports are changed. These rules are automatically
maintained by `gazelle`, which is run via `hack/update-bazel.sh`, but this is
still a source of friction.

[Autogazelle](https://github.com/bazelbuild/bazel-gazelle/tree/master/cmd/autogazelle)
is a new experimental tool which may reduce or remove the need for developers
to run `hack/update-bazel.sh`, but no work has yet been done to support it in
kubernetes/kubernetes.

### Code coverage support is incomplete for Go
Bazel and the Go rules have limited support for code coverage. Running something
like `bazel coverage -- //... -//vendor/...` will run tests in coverage mode,
but no report summary is currently generated. It may be possible to combine
`bazel coverage` with
[Gopherage](https://github.com/kubernetes/test-infra/tree/master/gopherage),
however.

### Kubernetes code generators are not fully supported
The make-based build system in kubernetes/kubernetes runs several code
generators at build time:
* [conversion-gen](https://github.com/kubernetes/code-generator/tree/master/cmd/conversion-gen)
* [deepcopy-gen](https://github.com/kubernetes/code-generator/tree/master/cmd/deepcopy-gen)
* [defaulter-gen](https://github.com/kubernetes/code-generator/tree/master/cmd/defaulter-gen)
* [openapi-gen](https://github.com/kubernetes/kube-openapi/tree/master/cmd/openapi-gen)
* [go-bindata](https://github.com/jteeuwen/go-bindata/tree/master/go-bindata)

Of these, only `openapi-gen` and `go-bindata` are currently supported when
building Kubernetes with Bazel.

The `go-bindata` generated code is produced by hand-written genrules.

The other code generators use special build tags of the form `//
+k8s:generator-name=arg`; for example, input files to the openapi-gen tool are
specified with `// +k8s:openapi-gen=true`.

`kazel` is used to find all packages that require OpenAPI generation, and then a
handwritten genrule consumes this list of packages to run `openapi-gen`.

For `openapi-gen`, a single output file is produced in a single Go package, which
makes this fairly compatible with Bazel.
All other Kubernetes code generators generally produce one output file per input
package, which is less compatible with the Bazel workflow.

The make-based build system batches up all input packages into one call to the
code generator binary, but this is inefficient for Bazel's incrementality, as a
change in one package may result in unnecessarily recompiling many other
packages.
On the other hand, calling the code generator binary multiple times is less
efficient than calling it once, since many of the generators parse the tree for
Go type information and other metadata.

One additional challenge is that many of the code generators add additional
Go imports which `gazelle` (and `autogazelle`) cannot infer, and so they must be
explicitly added as dependencies in the `BUILD` files.

Kubernetes has even more code generators than this limited list, but the rest
are generally run as `hack/update-*.sh` scripts and checked into the repository,
and so are not immediately needed for Bazel parity.

## Contacts
For help or discussion, join the [#bazel](https://kubernetes.slack.com/messages/bazel)
channel on Kubernetes Slack.
