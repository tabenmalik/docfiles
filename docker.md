# Docker

## Slim Docker Images

Slim container images make for faster builds,
less disk usage, and less bandwidth usage. Over
large scales, slim images can make a difference.
Here's a few techniques for keeping images slim.

### Remove unnecessary caches

If the cache is not necessary for the running of an image
then delete the cache. Make sure to clean up the cache
in the same `RUN` statement that the cache is generated,
otherwise the cache will remain in the image in an intermediate
layer.

Here's a few known tools that generate caches and how to delete
or avoid the cache:

* apt: `apt-get clean && rm -r /var/lib/apt/lists/*`
* pip: `pip install --no-cache-dir`
* yum: `yum clean all`
* dnf: `dnf clean all`
* conda: `conda clean --all --yes`

### Use multi-stage builds

Multi-stage builds allow for selectively copying artifacts
from one stage to another. For example, one stage can install
build dependencies and build a package, then the second stage
copies the built package without installing the build dependencies.

Example of building CPython in one stage and installing the build in a second stage:
```dockerfile
# build cpython
FROM ubuntu:22.04 AS CPythonBuild
WORKDIR /cpython
RUN : \
	&& echo deb-src http://archive.ubuntu.com/ubuntu/ jammy main >> /etc/apt/sources.list \
	&& apt-get update \
	&& DEBIAN_FRONTEND=noninteractive apt-get build-dep -y --no-install-recommends \
		python3 \
	&& DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
		git \
		pkg-config \
		zlib1g-dev \
	&& git init \
	&& git remote add origin https://github.com/python/cpython \
	&& git fetch --depth=1 \
	&& git switch main \
	&& ./configure \
	&& make \
	&& apt-get clean \
	&& rm -r /var/lib/apt/lists/* \
	&& :

# fresh image that installs built python
FROM ubuntu:22.04 as CPython
COPY --from=CPythonBuild /cpython/python /cpython/python
RUN /cpython/python -c "print('hello world!')"
```

Looking at the resulting image sizes, the CPython image is smaller than CPythonBuild image
due to the separate staging and selective copying.

```console
% docker image ls
REPOSITORY                TAG         IMAGE ID      CREATED        SIZE
localhost/cpython         316         1a08435e1659  2 minutes ago  118 MB
<none>                    <none>      8e321dd21b24  2 minutes ago  1.1 GB
```

## Style

These styling rules are not strict but what I personally use
to keep dockerfiles readable, grep-able, and git friendly.

### Trailing noop command

Use bash's `:` command to end compound `RUN` statements in
dockerfiles. An ending `:` command allows for inserting
new statements without generating a merge conflict (due
to the required newline escape).

The `:` bash built-in is a noop command equivalent to the
`true` command.

Example usage:
```dockerfile
RUN : \
    && command1 \
    && command2 \
    && :
```

Adding a new line before the the trailing `:`:
```dockerfile
RUN : \
    && command1 \
    && command2 \
    && inserted-command \
    && :
```

References:

* [anthony explains: dockerfile RUN : \ && syntax](https://youtu.be/BdxdRlTnPEE)

### Single line statements

Each command should be on its own line. Long commands can be broken
up into multiple lines with additional indentation.
Commands that list packages should have each package
on its own line and alphabetized.

```dockerfile
RUN : \
	&& command1 \
	&& download-pkgs \
		abc-pkg \
		hello-world \
		howdy-texas \
	&& long-command-with-many-flags \
		--flag1 \
		--flag2 \
		--flagN \
	&& comand-with-pipe | \
		second-command --process | \
		final-command \
	&& :
```
