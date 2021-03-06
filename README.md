# Cilium Dependency Packaging

This repository contains build definitions for a number of images that are components of the official and development images of Cilium.

The builds are currently hosted in GitHub Actions, but can be ported to any other container-based CI system.

Portability between CI systems and ability to run localy is critical, that's why some of these images are perfered over pre-packaged GitHub Actions.
Also, pre-package action often download dependencies on-the-fly, potentially increasing build times and causing flakiness. Registry is a network
resource, which could be unreliable at times, hower it can be mirrored easily, unlike GitHub releases.
Some of the image do depend on GitHub releases or other HTTP blol storage providers, but there is no easy way around that, as the only alrearnative
would be to build all of the dependencies from source, which is not feasible.

## Images

### [`images/maker`](images/maker/Dockerfile)

This image consists of core tools used for building all other images, which include `bash`, `make` and `docker` (with [`buildx`](https://github.com/docker/buildx))
and [`crane`](https://github.com/google/go-containerregistry/blob/master/cmd/crane).
This image enables using latest BuildKit features without depending on whatever Docker daemon/client CI host provides.
Since `buildx` runs a BuildKit daemon inside a container, it's largely independent of what version of Docker daemon it runs on.

This image also includes a secure credentials helper - [`docker-credential-env`](http://github.com/errordeveloper/docker-credential-env),
which prevents having to use `docker login` which stores a plain text token in `${DOCKER_CONFIG}/config.json`.

### [`images/kube-test`](images/kube-test/Dockerfile)

This image primarily provides `kubectl`, `helm`, `docker` and `kind`. The image is structurally similar to `maker`, and it also
include `docker`, however the purpose is different and it is important to keep the size of `maker` relatively small.
Also, it's unlikely that `maker` image will be updated as often as the `kube-test`.

### [`images/compiler`](images/compilers/Dockerfile)

This image consists of compilers and libraries needed to build other images for `amd64` and `arm64`.

It also includes multiple Bazel version to enable building different version of Istio and Envoy.

### [`images/bpftool`](images/bpftool/Dockerfile)

This image builds `bpftool` binary for `amd64` and `arm64` using a cross-compiler. The resulting image has only one file -
`/bin/bpftool`, it is a proper multi-platform image. The binary is dynamically linked to Ubuntu 20.04 glibc and other dependencies.

This image is uses a recent version of `bpftool` from `bpf-next` Linux kernel tree.

### [`images/iproute2`](images/iproute2/Dockerfile)

This image builds `ip` and `tc` binaries for `amd64` and `arm64` using a cross-compiler. The resulting image has only two files -
`/bin/ip` and `/bin/tc`, it is a proper multi-platform image. The binaries are dynamically linked to Ubuntu 20.04 glibc and other
dependencies.

This image is uses [a fork of `iproute2`](https://github.com/cilium/iproute2), it has features that Cilium relies on.

### [`images/llvm`](images/llvm/Dockerfile)

This image builds `llc` and `clang` binaries for `amd64` and `arm64` using a cross-compiler. The resulting image has only two
files - `/bin/llc` and `/bin/clang`, it is a proper multi-platform image. The binaries are dynamically linked to Ubuntu 20.04 glibc
and other dependencies.

This image is a custom BPF-only distribution of LLVM.

### [`images/tester`](images/tester/Dockerfile)

This image contains a [simple Go program](images/tester/cst/main.go), which is a minimal version of [`container-structure-test`](https://github.com/GoogleContainerTools/container-structure-test).
It's adapted to run inside a container build context more easily then the original `container-structure-tests`.

Here is how testing is accompilshed in the `llvm` image:
- [`images/llvm/Dockerfile`](https://github.com/cilium/image-tools/blob/3686e2885e854242f8835d6edfc7413dd7c4c476/images/llvm/Dockerfile#L25-L27)
- [`images/llvm/test/spec.yaml`](https://github.com/cilium/image-tools/blob/3686e2885e854242f8835d6edfc7413dd7c4c476/images/llvm/test/spec.yaml)

## Usage

### Making changes

All images get automatic tags based on checked-in contents of image subdirectory. At any point in git history of a subdirectory
there exists a unique [git tree object hash](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects), that is what's used for
image tags.

As the result of this, following stands:

- image build definitions can be obtained with `git show <tag>`
- image build is defined by contents of a directory
- when changes are committed to image directory, new tag is generated
    - if there is a new tag, image is rebuilt and pushed with that new tag

This does not cater for reproducible builds, however it serves as basis for reliable builds, especially when following rules
are also applied to any build definitions:

- all `FROM` statements use digests (use `scripts/get-image-digest.sh`)
- any system packages are installed in a separate image that is references by a digests (that's how `images/compilers` is designed)
    - pining system packages can be quite laborious, especially because most of the time what you want is latest that the distribution offers,
      so what's much easier to let the package manager get the latest and then pin down the result by digest, so every time there is a change
      in underlying system packages, that is explicitly recorded by change of digest in each image that uses the base image

Be sure to use `make lint`, which will run [`shellcheck`](https://github.com/koalaman/shellcheck) and [`hadolint`](https://github.com/hadolint/hadolint).

For details of how this works, see the following:

- [`Makefile`](Makefile)
- [`scripts/build-image.sh`](`scripts/build-image.sh`)
- [`scripts/make-image-tag.sh`](scripts/make-image-tag.sh).
- [`images/maker`](images/maker/Dockerfile)
- [`images/compilers`](images/compilers/Dockerfile)

### Building Locally

One should be able to build images locally as long as they have Docker installed with [`buildx` plug-in](https://docs.docker.com/buildx/working-with-buildx/).

### Updating `images/{maker,compilers}`

When you have dependencies that need to be added to these image before using them in one of the other images, e.g. if you need to add a system
library in `compilers` image that will be used for compiling something else, you should make a PR to update `compilers` first.
However, that's only required for full integration, and you can build images locally if you prefer, you can also push them to your own Docker Hub
account or whatever is your preferred registry.

When changes to these images are merged into master, builds should run and push new images to each of the registries.
Once new images are out, a PR will be required to update all dependent images, please use the following commands to
make updates and commit the resulting changes:

> NOTE: You can only use the `update-*-image` make targets when you are _not building localy_. For example, if you have built a new `compilers`
> locally, or perahps even pushed it your Docker Hyb account, and you want to consume this new version to build new `llvm` image - you need to uptade
> `images/llvm/Dockerfile` manually, as the `scripts/update-*-images.sh` is not capable of handling this.

- `make update-maker-image`
- `make update-compilers-image`
