![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)
![Release](https://img.shields.io/badge/release-v1.0-blue.svg)

# mkdocker

mkdocker is a lightweight system designed to make Docker image creation as simple and error‑free as possible. Its goal is to minimize the number of lines in a Dockerfile and reduce the likelihood of mistakes.

It consists of:
- a Bash script, `mkdocker`, used instead of `docker build ...`,
- a base image, `gmtscience/mamba`, which combines a micromamba image with helper scripts,
- and a set of Dockerfiles containing a few meta‑commands (see below) that simplify image creation.

## usage

Clone the repository, install `mkdocker` in your PATH, and optionally set the `REGISTRY` environment variable to either your Docker Hub namespace (for us, `gmtscience`) or a full private registry. The built image will be pushed to `$REGISTRY/<dockerfile name>`.

Create a folder for your Dockerfiles, e.g. `/my/dockers/`.

Usage:
```sh
cd /my/dockers
# edit a dockerfile as usual (see examples below)
vi somedocker
mkdocker somedocker
```

## features

- Dockerfiles are not named `Dockerfile` anymore; instead, they take the name of the image they define.
- The system builds on micromamba: micromamba is extremely efficient for Conda‑based environments, but it relies on a dedicated (non‑root) user, which complicates installing system packages. The base `mamba` image solves this issue.
- The base image also provides simple helper commands beginning with `_`, which wrap the underlying tools with Docker‑friendly optimizations (e.g., reduced image size, automatic micromamba activation):
  - `_conda` replaces `conda` (install only): activates the micromamba environment and cleans up afterward,
  - `_pip` replaces `pip` (install only): uses the Conda Python environment and applies Docker optimization tricks,
  - `_apt` replaces `apt` (install only): automatically adds `-y` and applies cleanup steps,
  - `_git` replaces `git` (clone only): adds sensible defaults for Docker builds (`--single-branch --depth 1`), disables unnecessary SSH checks, and accelerates cloning.

Another base image is provided: `rustbuilder`, intended for multi‑stage builds. It includes the `_cargo` command (build only), which activates the Rust environment and uses safe build defaults.

## extra Dockerfile commands

`mkdocker` scans the Dockerfile for specific hash‑prefixed meta‑commands and applies corresponding build‑time behaviors:

- `#tag <tag>` adds the specified `<tag>` to the built image (in addition to `latest`, which is always included),
- `#nopush` disables pushing the image to a registry (pushing is enabled by default),
- `#registry <registry>` overrides the default registry (which otherwise comes from `$REGISTRY`).  
  If neither a registry nor `#nopush` is set, a warning is shown.
- `#usessh` enables SSH agent forwarding for private repository cloning (see below).

Each meta‑command must appear at the beginning of the line to be recognized.

Aside from these commands, the Dockerfile is completely standard.

## #usessh and git clone

The `#usessh` directive ensures that your SSH agent is available during the build so that `_git` can clone private repositories.  
Your `RUN` command must include `--mount=type=ssh`. For example:

```docker
#usessh
RUN --mount=type=ssh _git clone -b mytag ssh://git@my.private.repo/mycode
```

This allows the Docker build (triggered via `mkdocker`) to reuse your local SSH credentials without storing SSH keys inside the image or prompting for a password.

The SSH agent is automatically shut down once the build finishes.

`#usessh` and all other meta‑commands can appear anywhere in the Dockerfile (as long as they start the line), since they are parsed before the build begins.

## multi-stage build

Multi‑stage builds are strongly recommended when compilation is involved. They pair well with `_git` and `_cargo`.  
Example:

```docker
FROM gmtscience/rustbuilder
#usessh
RUN --mount=type=ssh _git clone -b 'v1.0' ssh://gitolite@git.gmt.bio/pipeline/pipeline.git /pipeline/
RUN cd /pipeline/counter && _cargo build

FROM gmtscience/mamba
COPY --from=0 /metagen/counter/target/release/pipeline-counter /usr/local/bin/pipeline-counter
#tag 1.0
```

Explanation:
- The first stage performs the build. It uses the Rust builder image, enables SSH cloning, and compiles the project.
- The second stage produces the final image, copying only the resulting binary via `COPY --from=0`.

## examples

### conda‑based environment

`mkdocker` was originally created to make this scenario trivial:

```docker
FROM gmtscience/mamba

RUN _conda install fastp=0.23.4
```

This produces a functional fastp image. While a multi‑stage build could produce a smaller image, the simplicity here is usually worth it:
- `_conda install` behaves like normal Conda but supports multiple packages in a single line,
- the more packages you install, the more advantageous this approach becomes.

### simple download

A very lazy fastp image:

```docker
FROM gmtscience/mamba

RUN curl -L http://opengene.org/fastp/fastp.0.23.4 -o /usr/local/bin/fastp && chmod a+x /usr/local/bin/fastp
```

### multi‑stage example from source

The most complex approach, used only when no prebuilt binary or Conda package is available:

```docker
FROM gmtscience/basebuilder

RUN _apt install nasm yasm
RUN _git clone -b v2.31.0 https://github.com/intel/isa-l.git /isa-l
RUN cd /isa-l && ./autogen.sh && ./configure --prefix=/usr --libdir=/usr/lib64 && make install

RUN _git clone -b v1.20 https://github.com/ebiggers/libdeflate.git /libdeflate
RUN cd /libdeflate && cmake -B build && cmake --build build && cmake --install build

RUN _git clone -b v0.23.4 https://github.com/OpenGene/fastp.git /fastp
RUN cd /fastp && make static

FROM gmtscience/mamba
COPY --from=0 /fastp/fastp /usr/local/bin/
```
