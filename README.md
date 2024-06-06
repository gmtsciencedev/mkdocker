# mkdocker

mkdocker is a rustic hack designed to make docker creation as simple as possible. It tries to lower the number of lines in a Dockerfile to a minimum so as to reduce the number of errors.

## usage

clone the repo, install `mkdocker` in your path, optionnally set the `REGISTRY` shell variable either to the name of your Docker hub namespace (for us, that is `gmtscience`, or to a complete private registry, the docker image will pushed to `$REGISTRY/<name of the docker file>`). Create a folder where to put all your dockerfiles: `/my/dockers/`.

Usage is then:
```sh
cd /my/dockers
# now edit a dockerfile as usual, see below the examples
vi somedocker
mkdocker somedocker
```

## features

- by default the Dockerfiles are no longer called `Dockerfile` but bear the name of the image they code for.
- building up on micromamba: mamba/micromamba is a killer environment that we are really glad to use. It makes Anaconda install so quick and lean. However it relies on a specific user and not root which makes difficult the installation of distribution packages for instance. Thus mkdocker relies on a base image, `mamba` which solve this particular issue.
- the base image also provides some simple commands that begin with an underscore to replace base commands while adding specific optimization for docker (to reduce image size notably or activate environment):
  - `_conda` replace `conda` (only install subcommand supported), it adds micromamba environment activation and does conda cleaning after install,
  - `_pip` replace `pip` (only install subcommand supported), it uses python from conda environment (thus activate micromamba environment) and limits impact on image size with well known docker tricks,
  - `_apt` replace `apt` (only install subcommand supported), it adds the `-y` option and limits impact on image size with well known docker tricks,
  - `_git` replace `git` (only clone subcommand supported) (see `#usessh` below), it adds some options to suppress SSH checking which make no sense in a docker building context, it adds several options like `--single-branch --depth 1` which makes git a lot quicker and limit the size of downloaded source.

Another base image is also provided, the `rustbuilder` image, which provides a suitable environment for rust building (and is meant to be use in multi-stage build, see below). This particular image (built upon the base image) provides the `_cargo` command (only build subcommand supported) which activate the cargo environment and use some safe building defaults.

## extra Dockerfile commands

`mkdocker` looks for specific comment to add those some tricks to Dockerfile so as to make extrabuilding scripts unnecessary:

- `#tag <tag>` add the <tag> building tag when building (latest is always automatically added),
- `#nopush` change the default option which is to push to remote registry to no push,
- `#registry <registry>` set the remote registry to a custom option (the default registry is set via shell variable $REGISTRY) (if neither is set a warning will occur, set `#nopush` to remove the warning, meaning it is intendended as an image not to be pushed)
- `#usessh` is a hack for private repository, see below.

NB the line must start with the command otherwise it won't trigger

Other than that, the file is just a plain Dockerfile, so all other docker specifics can be used.

## #usessh and git clone

The `#usessh` commands ensure that your SSH agent is up so as to transfer your SSH settings and access to the docker `_git` (see above) command. You must also start your RUN command with the `--mount=type=ssh`, see the example below so that it works. It also shut down the agent once the docker image is built.

So basically you should have the two line in your Dockerfile:
```docker
#usessh
RUN --mount=type=ssh _git clone -b mytag ssh://git@my.private.repo/mycode
```

This will enable to use your user SSH enabled authentication to be used within docker, during the docker build phase only, i.e. when you run the `mkdocker` command. This removes the need for adding an SSH key in the build image or ask for a password during build.

NB the `#usessh` or any other mkdocker hash instruction may be anywhere in the script, they are evaluated before the docker build so it does not matter as long as the line starts with the has instruction.

## multi-stage build

Multi-stage build is advised in docker whenever a compilation step is involved, so usually it should be combined with a _git command 
Here is a fake example derived from a real docker from our own collection:

```docker
FROM gmtscience/rustbuilder
#usessh
RUN --mount=type=ssh _git clone -b 'v1.0' ssh://gitolite@git.gmt.bio/pipeline/pipeline.git /pipeline/
RUN cd /pipeline/counter && _cargo build

FROM gmtscience/mamba
COPY --from=0 /metagen/counter/target/release/pipeline-counter /usr/local/bin/pipeline-counter
#tag 1.0
```

If you do not know about multi-stage docker build, here is a small description of what happens here:
- there are two FROM instruction in the dockerfile, which is the signature of multi-stage build. The first paragraphe describe the building docker. It use the `#usessh` instruction that wa just explained above and the _cargo helper to ease Rust building from the the rustbuilder image,
- in the second paragraph, the final image is created, with a unique `COPY` instruction which copy the binary produced before from the building docker (`--from=0`).

## examples

### conda derived environment

`mkdocker` was invented to make this as simple as possible, so this one is a two liner:

```docker
FROM gmtscience/mamba

RUN _conda install fastp=0.23.4
```

This is a fastp docker image - it may not be the most optimized fastp image (a multi-stage build would be slimer) but it's certainly the easiest code you'll find to create one. Three remarks:
- `_conda install` is just like any conda install, you can have a long list of packages all in one line, specifying or not versions like with our `=0.23.4` option,
- the more packages you'll need, the more this conda approach makes sense and makes unlikely the fact that you'll gain much with a multi-stage build.

### simple download 

Another fastp image, even lazier:
```docker
FROM gmtscience/mamba

RUN curl -L http://opengene.org/fastp/fastp.0.23.4 -o /usr/local/bin/fastp && chmod a+x /usr/local/bin/fastp
```

### multi-stage approach

The most complexe, from source approach, clearly not the one we recommand for fastp but maybe the only available options in some cases.

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

