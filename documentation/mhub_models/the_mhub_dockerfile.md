# The MHub Dockerfile

Every MHub model is bundled with all dependencies and ressources in a Docker container.
This document explains how to write your MHub-compliant `Dockerfile`.

***NOTE:** This document is still work in progress. More examples and references will be added in the near future.**

You can find examples of MHub-compliant Dockerfiles in our [Models Repository](https://github.com/MHubAI/models/blob/main/base/dockerfiles/Dockerfile).

## Start from our official Base image

All MHub models are based on our official MHub Base image `mhubai/base:latest` that is available on [DockerHub](https://hub.docker.com/). The source code of the base image's Dockerfile can be found in our [Models Repository](https://github.com/MHubAI/models/blob/main/base/dockerfiles/Dockerfile).

To start writing your own `Dockerfile`, you must first start from our base image. Therefore, the first line of your `Dockerfile` must be:

```dockerfile
FROM mhubai/base:latest

# ...
```

## Install your dependencies

Next, install all your system dependencies, e.g. python packages.

## Clone your model's repository

Then you need to clone your [model folder](model_folder_structure.md) to your Dockerfile with a [sparse checkout](https://git-scm.com/docs/git-sparse-checkout) from the MHub [Models Repository](https://github.com/MHubAI/models/).

We prepared a build utility script that helps with this step. Add the following three lines to your Dockerfile and replace `my_model` with the name of your model. The `import_mhub_model.sh` script automatically loads your model structure from our models repository.

```dockerfile
# Import the MHub model definiton
ARG MHUB_MODELS_REPO
RUN buildutils/import_mhub_model.sh my_model ${MHUB_MODELS_REPO}
```

As your model will, of course, not be available on our models repository unless your contribution is accepted, this won't build. The lines above, therefore, introduce a build argument, `MHUB_MODELS_REPO`, that you can use to specify your repository and branch at build time. As shown below, you can then specify `--build-arg` with the `docker build` command to set `MHUB_MODELS_REPO` to the URL of your fork of our models repository and optionally specify the branch name separated with `::` from the URL.

```bash
docker build --build-arg MHUB_MODELS_REPO=https://github.com/your_username/models-fork::branch -t dev/my_model:latest.
```

## Clone Source Code

Now clone the source code of your model into the container. To ensure that no changes are made to external resources without forcing a rebuild of the model, we insist that you check out a fixed revision by explicitly specifying the commit that will be cloned into the Dockerfile.

## Download Weights

You need to download all the resources in your Dockerfile in advance. You may have a routine that downloads additional resources, such as model weights, on demand when your model first runs. However, since we are working in a container environment, every time a container is spun up is technically a first-time execution. We recommend compensating for on-demand downloads with a pre-download in the Docker file. This will result in a longer build time and larger image size, but the model can be run immediately and the MHub modules can run without an internet connection. This is especially important since we recommend that all users run MHub models with the `--network none` option.

## Set the Entrypoint

All MHub containers must provide the following entrypoint and cmd:

```dockerfile
ENTRYPOINT ["python3", "-m", "mhubio.run"]
CMD ["--workflow", "default"]
```

## The Standalone Concept

All MHub Dockerfiles must be *standalone*, i.e., they can be built without additional local dependencies. Therefore, use of the `COPY` or `ADD` statements is prohibited. Instead, download or clone the online resources at build time and use Docker mounts during development.
