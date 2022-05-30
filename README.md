# clj-nix

Nix helpers for Clojure projects

STATUS: alpha. Please leave feedback.

## Table of contents

- [Introduction](#introduction)
- [Usage](#usage)
  - [Generate lock file](#generate-lock-file)
  - [API](#api)
- [Tutorial](#tutorial)
- [Similar projects](#similar-projects)

## Introduction

The main goal of the project is to reduce the friction between Clojure and Nix.
Nix is a great tool to build and deploy software, but Clojure is not well
supported in the Nix ecosystem.

`clj-nix` tries to improve the situation, providing Nix helpers to interact with
Clojure projects

The main difficulty of packaging a Clojure application with Nix is that the
derivation is restricted from performing any network request. But Clojure does
network requests to resolve the dependency tree. Some network requests are done
by Maven, since Clojure uses Maven under the hood. On the other hand, since
[git deps](https://clojure.org/news/2018/01/05/git-deps) were introduced,
Clojure also access the network to resolve the git dependencies.

A common solution to this problem are **lock files**. A lock file is a snapshot
of the entire dependency tree, usually generated the first time we install the
dependencies. Subsequent installations will use the lock file to install exactly
the same version of every dependency. Knowing beforehand all the dependencies,
we can download and cache all of them, avoiding network requests during the
build phase with Nix.

Ideally, we could reuse a lock file generated by Maven or Clojure itself, but
lock files are not popular in the JVM/Maven ecosystem. For that reason,
`clj-nix` provides a way to create a lock file from a `deps.edn` file. Creating
a lock file is a prerequisite to use the Nix helpers provided by `clj-nix`

**GOALS**:

- Create a binary from a clojure application
- Create an optimized JDK runtime to execute the clojure binary
- Create GraalVM native images from a clojure application
- Simplify container creation for a Clojure application
- Run any arbitrary clojure command at Nix build time (like `clj -T:build` or
  `clj -M:test`)

## Usage

This project requires [Nix Flakes](https://nixos.wiki/wiki/Flakes)

### New project template

```bash
nix flake new --template github:jlesquembre/clj-nix ./my-new-project
cd ./my-new-project
git init
git add .
```

Remember that with flakes, only the files tracked by git are recognized by Nix.

Templates are for new projects. If you want to add `clj-nix` to an existing
project, I suggest just copy the parts you need from the template (located here:
[clj-nix/templates/default](https://github.com/jlesquembre/clj-nix/tree/main/templates/default))

### Generate lock file

As mentioned, a lock file must be generated in advance:

```bash
nix run github:jlesquembre/clj-nix#deps-lock
```

That command generates a `deps-lock.json` file in the current directory.
Remember to re-run it if you update your dependencies.

### API

Derivations:

- [mkCljBin](#mkcljbin): Creates a clojure application
- [customJdk](#customjdk): Creates a custom JDK with jlink. Optionally takes a
  derivation created with `mkCljBin`. The intended use case is to create a
  minimal JDK you can deploy in a container (e.g: a Docker image)
- [mkGraalBin](#mkgraalbin): Creates a binary with GraalVM from a derivation
  created with `mkCljBin`

**NOTE**: Extra unknown attributes are passed to the `mkDerivation` function,
see [mkCljBin](#mkcljbin) section for an example about how to add a custom check
phase.

Helpers:

- [mkCljCli](#mkcljcli): Takes a derivation created with `customJdk` and returns
  a valid command to launch the application, as a string. Useful when creating a
  container.

#### mkCljBin

Creates a Clojure application. Takes the following attributes (those without a
default are mandatory, extra attributes are passed to **mkDerivation**):

- **jdkRunner**: JDK used at runtime by the application. (Default: `jdk`)

- **projectSrc**: Project source code.

- **name**: Derivation and clojure project name. It's recommended to use a
  namespaced name. If not, a namespace is added automatically. E.g. `foo` will
  be transformed to `foo/foo`

- **version**: Derivation and clojure project version. (Default: `DEV`)

- **main-ns**: Main clojure namespace. A `-main` function is expected here.

- **buildCommand**: Command to build the jar application. If not provided, a
  default builder is used:
  [build.clj](https://github.com/jlesquembre/clj-nix/blob/main/src/cljnix/build.clj).
  If you provide your own build command, clj-nix expects that a jar will be
  generated in a directory called `target`

**Example**:

```nix
mkCljBin {
  jdkRunner = pkgs.jdk17_headless;
  projectSrc = ./.;
  name = "me.lafuente/clj-tuto";
  version = "1.0";
  main-ns = "demo.core";

  buildCommand = "clj -T:build uber";

  # mkDerivation attributes
  doCheck = true;
  checkPhase = "clj -M:test";
}
```

**Outputs**:

- **out**: The application binary
- **lib**: The application jar

#### customJdk

Creates a custom JDK runtime. Takes the following attributes (those without a
default are mandatory):

- **jdkBase**: JDK used to build the custom JDK with jlink. (Default:
  `nixpkgs.jdk17_headless`)

- **cljDrv**: Derivation generated with `mkCljBin`.

- **name**: Derivation name. (Default: `cljDrv.name`)

- **version**: Derivation version. (Default: `cljDrv.version`)

- **jdkModules**: Option passed to jlink `--add-modules`. If null, `jeps` will
  be used to analyze the `cljDrv` and pick the necessary modules automatically.
  (Default: `null`)

- **locales**: Option passed to jlink `--include-locales`. (Default: `null`)

**Example**:

```nix
customJdk {
  jdkBase = pkgs.jdk17_headless;
  name = "myApp";
  version = "1.0.0";
  cljDrv = myCljBinDerivation;
  locales = "en,es";
}
```

**Outputs**:

- **out**: The application binary, using the custom JDK
- **jdk**: The custom JDK

#### mkGraalBin

Generates a binary with GraalVM from an application created with `mkCljBin`.
Takes the following attributes (those without a default are mandatory):

- **cljDrv**: Derivation generated with `mkCljBin`.

- **graalvm**: GraalVM used at build time. (Default:
  `nixpkgs.graalvmCEPackages.graalvm17-ce`)

- **name**: Derivation name. (Default: `cljDrv.name`)

- **version**: Derivation version. (Default: `cljDrv.version`)

- **extraNativeImageBuildArgs**: Extra arguments to be passed to the
  native-image command. (Default: `[ ]`)

- **graalvmXmx**: XMX size of GraalVM during build (Default: `"-J-Xmx6g"`)

**Example**:

```nix
mkGraalBin {
  cljDrv = myCljBinDerivation;
}
```

An extra attribute is present in the derivation, `agentlib`, which generates a
script to help with the generation of a reflection config file

#### mkCljCli

Returns a string with the command to launch an application created with
`customJdk`. Takes the following attributes (those without a default are
mandatory):

**jdkDrv**: Derivation generated with `customJdk`

**java-opts**: Extra arguments for the Java command (Default: `[]`)

**extra-args**: Extra arguments for the Clojure application (Default: `""`)

**Example**:

```nix
mkCljCli {
  jdkDrv = self.packages."${system}".jdk-tuto;
  java-opts = [ "-Dclojure.compiler.direct-linking=true" ];
  extra-args = [ "--foo bar" ];
}
```

## Tutorial

Source code for this tutorial can be found here:
https://github.com/jlesquembre/clj-demo-project

### Init

There is a template to help you start your new project:

```bash
nix flake new --template github:jlesquembre/clj-nix ./my-new-project
```

For this tutorial you can clone the final version:

```bash
git clone git@github.com:jlesquembre/clj-demo-project.git
```

First thing we need to do is to generate a lock file:

```bash
nix run github:jlesquembre/clj-nix#deps-lock
git add deps-lock.json
```

**NOTE**: The following examples assume that you cloned the demo repository, and
you are executing the commands from the root of the repository. But with Nix
flakes, it's possible to point to the remote git repository. E.g.: We can
replace `nix run .#foo` with `nix run github:/jlesquembre/clj-demo-project#foo`

### Create a binary from a Clojure application

First, we create a new package in our flake:

```nix
clj-tuto = cljpkgs.mkCljBin {
  projectSrc = ./.;
  name = "me.lafuente/cljdemo";
  main-ns = "demo.core";
};
```

Let's try it:

```bash
nix build .#clj-tuto
./result/bin/clj-tuto
# Or
nix run .#clj-tuto
```

Nice! We have a binary for our application. But how big is our app? We can find
it with:

```bash
nix path-info -sSh .#clj-tuto
# Or to see all the dependencies:
nix path-info -rsSh .#clj-tuto
```

Um, the size of our application is `1.3G`, not ideal if we want to create a
container. We can use a headless JDK to reduce the size, let's try that:

```nix
clj-tuto = cljpkgs.mkCljBin {
  projectSrc = ./.;
  name = "me.lafuente/cljdemo";
  main-ns = "demo.core";
  jdkRunner = pkgs.jdk17_headless;
};
```

```bash
nix build .#clj-tuto
nix path-info -sSh .#clj-tuto
```

Good, now the size is `703.9M`. It's an improvement, but still big. To reduce
the size, we can use the `customJdk` helper.

### Create custom JDK for a Clojure application

We add a package to our flake, to build a customized JDK for our Clojure
application:

```nix
jdk-tuto = cljpkgs.customJdk {
  cljDrv = self.packages."${system}".clj-tuto;
  locales = "en,es";
};
```

```bash
nix build .#jdk-tuto
nix path-info -sSh .#jdk-tuto
```

Not bad! We reduced the size to `96.3M`. That's something we can put in a
container. Let's create a container with our application.

### Create a container

Again, we add a new package to our flake, in this case it will create a
container:

```nix
clj-container =
  pkgs.dockerTools.buildLayeredImage {
    name = "clj-nix";
    tag = "latest";
    config = {
      Cmd = clj-nix.lib.mkCljCli self.packages."${system}".jdk-tuto { };
    };
  };
```

```bash
nix build .#clj-container
nix path-info -sSh .#clj-container
```

The container's size is `52.8M`. Wait, how can be smaller than our custom JDK
derivation? There are 2 things to consider.

First, notice that we used the `mkCljCli` helper function. In the original
version, our binary is a bash script, so `bash` is a dependency. But in a
container we don't need `bash`, the container runtime can launch the command,
and we can reduce the size by removing `bash`

Second, notice that the image was compressed with gzip.

Let's load and execute the image:

```bash
docker load < result
docker run -it --rm clj-nix
docker images
```

Docker reports an image size of `99.2MB`

### Create a native image with GraalVM

If we want to continue reducing the size of our derivation, we can compile the
application with GraalVM. Keep in mind that size it's not the only factor to
consider. There is a nice slide from the GraalVM team, illustrating what
technology to use for which use case:

![GraalVM performance](docs/graal-performance.jpeg)

(The image was taken from a
[tweet by Thomas Würthinger](https://twitter.com/thomaswue/status/1145603781108928513))

For more details, see:
[Does GraalVM native image increase overall application performance or just reduce startup times?](https://stackoverflow.com/a/59488814/799785)

Let's compile our Clojure application with GraalVM:

```nix
graal-tuto = cljpkgs.mkGraalBin {
  cljDrv = self.packages."${system}".clj-tuto;
};
```

```bash
nix build .#graal-tuto
./result/bin/clj-tuto
nix path-info -sSh .#graal-tuto
```

The size is just `43.4M`.

We can create a container from this derivation too:

```nix
graal-container =
  let
    graalDrv = self.packages."${system}".graal-tuto;
  in
  pkgs.dockerTools.buildLayeredImage {
    name = "clj-graal-nix";
    tag = "latest";
    config = {
      Cmd = "${graalDrv}/bin/${graalDrv.name}";
    };
  };
```

```bash
docker load < result
docker run -it --rm clj-graal-nix
```

In this case, the container image size is `45.3MB`, aproximately half the size
of the custom JDK image.

<!-- ## GraalVM real examples -->

<!-- Currently, in nixpkgs we need an uberjar to generate a native image with -->
<!-- GraalVM. But such uberjar is generated outside of nix, usually in a -->
<!-- non-deterministic way. It would be better if we relay only in the source code, -->
<!-- not every project generates an uberjar (but some will generate one -->
<!-- [if you ask politely](https://github.com/clj-kondo/clj-kondo/issues/414) :-) ) -->

<!-- The main difference between this project and the current nixpkgs approach is -->
<!-- that with `clj-nix` we should be able to generate a binary directly from the -->
<!-- code. -->

<!-- ### clj-kondo -->

<!-- ``` -->
<!-- nix run github:/jlesquembre/clj-demo-project#clj-kondo -->
<!-- ``` -->

## Similar projects

- [dwn](https://github.com/webnf/dwn)
- [clj2nix](https://github.com/hlolli/clj2nix)
- [mvn2nix](https://github.com/fzakaria/mvn2nix)
- [clojure-nix-locker](https://github.com/bevuta/clojure-nix-locker)
