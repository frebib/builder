[hub]: https://hub.docker.com/r/frebib/builder

[frebib/builder][hub]
=============
[![](https://images.microbadger.com/badges/image/frebib/builder.svg)](https://microbadger.com/images/frebib/builder)[![Docker Pulls](https://img.shields.io/docker/pulls/frebib/builder.svg)][hub][![Docker Stars](https://img.shields.io/docker/stars/frebib/builder.svg)][hub]

A docker image that builds, tests and pushes docker images from code repositories.
This image is a continuation of [tutum/builder](https://github.com/tutumcloud/builder), updated to use [library/docker](https://github.com/docker-library/docker) base image for Docker-in-Docker based on the [Alpine Linux project](https://hub.docker.com/_/alpine/).


# Usage

## Build from local folder

Run the following docker command in the folder that you want to build and push:

	docker run --rm -it --privileged -v $HOME/.docker:/.docker:ro -v $(pwd):/app frebib/builder $IMAGE_NAME

Where:

* `$IMAGE_NAME` (optional) is the name of the image to build and push with an optional tag, i.e. `frebib/hello-world:latest`. If not specified, it will be built and tested, but not pushed. It can also be passed in as an environment variable `-e IMAGE_NAME=$IMAGE_NAME`.

This will use the `~/.docker/config.json` file which should be prepopulated with credentials by using `docker login <registry>` in the host. Alternatively, you can use `$USERNAME`, `$PASSWORD` as described below.


## Build from Git repository

Run the following docker command:

	docker run --rm -it --privileged -v $HOME/.docker:/.docker:ro -e GIT_REPO=$GIT_REPO -e USERNAME=$USERNAME -e PASSWORD=$PASSWORD -e DOCKERFILE_PATH=$DOCKERFILE_PATH frebib/builder $IMAGE_NAME

Where:

* `$GIT_REPO` is the git repository to clone and build, i.e. `https://github.com/frebib/docker-autobuilder.git`
* `$GIT_TAG` (optional, defaults to `master`) is the tag/branch/commit to checkout after clone, i.e. `master`
* `$DOCKERFILE_PATH` (optional, defaults to `/`) is the relative path to the root of the repository where the `Dockerfile` is present, i.e. `/`
* `$IMAGE_NAME` is the name of the image to create with an optional tag, i.e. `frebib/autobuilder:latest`
* `$USERNAME` is the username to use to log into the registry using `docker login`
* `$PASSWORD` is the password to use to log into the registry using `docker login`

If you want to use a SSH key to clone your repository, mount your private SSH key to `/root/.ssh/id_rsa` inside the container, by appending `-v ~/.ssh/id_rsa:/root/.ssh/id_rsa` to the `docker run` command above. Or you can use the environment variable below:
* `$GIT_ID_RSA` (optional) is the private SSH key to use when cloning the git repository (i.e. `-e GIT_ID_RSA="$(awk 1 ORS='\\n' ~/.ssh/id_rsa)"`)


## Build from compressed tarball

Run the following docker command:

	docker run --rm -it --privileged -v $HOME/.docker:/.docker:ro -e TGZ_URL=$TGZ_URL -e DOCKERFILE_PATH=$DOCKERFILE_PATH -e USERNAME=$USERNAME -e PASSWORD=$PASSWORD frebib/builder $IMAGE_NAME

Where:

* `$TGZ_URL` is the URL to the compressed tarball (.tgz) to download and build, i.e. `https://github.com/frebib/docker-hello-world/archive/v1.0.tar.gz`
* `$DOCKERFILE_PATH` (optional, defaults to `/`) is the relative path to the root of the tarball where the `Dockerfile` is present, i.e. `/docker-hello-world-1.0`
* `$IMAGE_NAME` is the name of the image to create with an optional tag, i.e. `frebib/hello-world:latest`
* `$USERNAME` is the username to use to log into the registry using `docker login`
* `$PASSWORD` is the password to use to log into the registry using `docker login`


# Tagging

Image tags can be statically added to an image via the **$IMAGE_TAGS** environment variable, with multiple tags being space separated, details explained in the [Environment Variables](#environment-variables) section.

Alternatively tags can be generated with the hook *hooks/tag* which is provided so tags can be generated with scripts for the built image. Each new tag should be on a separate line

Regardless of which method by which the tags are provided, the first one provided is used as the default tag for referring to the image in log output and saving the image to file. All tags are expected to be valid docker tags.


# Testing

If you want to test your app before building, create a `docker-compose.test.yml` file in your repository root with a service called `sut` which will be run for testing. You can specify another file name in `$TEST_FILENAME` if required. If that container exits successfully (exit code 0), the build will continue; otherwise, the build will fail and the image won't be built nor pushed.

Example `docker-compose.test.yml` file for a Django app that depends on a Redis cache:

	sut:
	  build: .
	  links:
	    - redis
	  command: python manage.py test
	redis:
	  image: tutum/redis
	  environment:
	    - REDIS_PASS=password

To speed up testing, you can replace `build: .` in your `sut` service with `image: this`, which is the name of the image that is built just before running the tests. This way you can avoid building the same image twice.


# Hooks

There is the possibility to run scripts before and after some of the build steps to set up your application as required. The following hooks are available (in this order):

* `hooks/post_checkout` (does not run if mounting `/app`)
* `hooks/tag` (to override the **$IMAGE_TAGS** variable)
* `hooks/pre_build`
* `hooks/build` (to override the default **build** step)
* `hooks/post_build`
* `hooks/pre_test`
* `hooks/test` (to override the default **test** step)
* `hooks/post_test`
* `hooks/pre_push`
* `hooks/push` (to override the default **push** step)
* `hooks/post_push`

Create a file in your repository in a folder called `hooks` with those names and the builder will execute them before and after each step.

# Environment Variables

The following environment variables are available for testing, when executing the docker-compose.test.yml file; and during the execution of hooks.

* `$GIT_BRANCH` which contains the name of the branch that is currently being tested
* `$GIT_TAG` which contains the branch/tag/commit being tested
* `$GIT_SHA1` which contains the commmit hash of the tag being tested
* `$IMAGE_NAME` which contains the name of the docker repository being built (if defined when launching the container)
* `$IMAGE_TAGS` which contains a static, space-delimited list of tags for the image
* `$GIT_CLONE_OPTS` which is the option passed to "git clone", default: `--recursive`
* `$SAVE` which when set `true` saves the tagged image to a */images/$repository/$image\_name$tag.tar*


# Notes

## Caching images for faster builds

If you want to cache the images used for building and testing, run the following:

	docker run --name builder_cache --entrypoint /bin/true frebib/builder

And then run your builds as above appending `--volumes-from builder_cache` to them to reuse already downloaded image layers.

## Adding repository credentials

If your tests depend on private images, you can pass their credentials either by mounting your local `.docker` folder inside the container appending `-v $HOME/.docker:/.docker:ro`, or by providing the contents of the docker configuration file `$HOME/.docker/config.json` via an environment variable called `$DOCKER_CONFIG`. eg `-e DOCKER_CONFIG="$(cat $HOME/.docker/config.json)"`

## Using the host docker daemon instead of docker-in-docker

If you want to use the host docker daemon instead of letting the container run its own, mount the host's docker unix socket inside the container by appending `-v /var/run/docker.sock:/var/run/docker.sock:rw` to the `docker run` command.

## Disable recursive strategy on git clone

In some cases, you may need set up credentials for the initialization github submodules, and it may make `git clone --recursive` fail. In such a case, you can disable the default `--recursive` strategy by setting `-e GIT_CLONE_OPTS=""` and do `git submodule init & git submodule update` manually in `hooks/post_checkout` or `hooks/pre_build`.
