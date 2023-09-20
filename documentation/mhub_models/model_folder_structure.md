# MHub Model Structure

All MHub models are organized in our [Models Repository](https://github.com/MHubAI/models/).
There we provide our base image under `/base` and each model has its own folder under `/models`.
The folder of the model is named after the [MHub name](model_json.md#name) of the model. Throughout this example, we consider a model named `my_model`.

This document explains how the model folders are structured on MHub and how you set them up.

## Model Folder Structure

You'll create a folder under `/models/my_model` for your model.
Inside your model's folder, create the following folders: `dockerfiles`, `config` and `utils` alongside a `README.md` and a `meta.json` file, each of which are explained in more detail below.

```text
/models/my_model
├── dockerfiles/
│   └── Dockerfile
├── config/
│   ├── default.yml
│   └── slicer.yml
├── utils/
│   └── ModelRunner.py
└── meta.json
```

## Dockerfiles

Create a single `Dockerfile` in the `/models/my_model/dockerfiles` folder. The `Dockerfile` is used to bundle your model's code together with all dependencies and ressources into a Docker container. More details on how to write your MHub compliant `Dockerfile` can be found in the [MHub Dockerfile documentation](the_mhub_dockerfile.md).

## Config

The configuration files for your model are stored in the folder `/models/my_model/config`. All MHub models share a similarly structured configuration file that bundles the configuration for all MHub modules in one place.

Each configuration file represents an executable workflow of an MHub model. You must always provide at least one default workflow `default.yml` that starts on DICOM data and generates DICOM data (e.g. DICOMSEG for segmentation models). However, you can provide additional workflows in separate configuration files that start on different data or use different configurations. We recommend to provide only configurations for the most general use cases that are suitable for most people.

Therefore, create at least one `default.yml` file in the configs folder. More information about the structure of the configuration file can be found [here](../mhubio/the_mhubio_config_file.md).

## Utils

Each MHub workflow, as described in your configuration file, consists of a list of modules that are executed in sequence. An overview with detailed explanations of all our core modules can be found [here](../mhubio/mhubio_modules.md). In addition to our core modules, you can also write your own modules. For example, you will always write a RunnerModule that wraps your AI pipeline, but you can also write additional pre- or post-processing modules, converters or report generators.

All custom modules are stored in the `/models/my_model/utils` folder. Start writing your `MyModelRunner` runenr module as explained [in this article](../mhubio/how_to_write_an_mhubio_module.md) and place it in a `MyModelRunner.py` file in the utils folder. It is important that you provide only one module class per file, name the file exactly like the class and place it in the utils folder so that MHub- IO can find and import your module automatically.

## Metadata

Each MHub model must provide a `meta.json` file in the model's root directory. This file contains all metadata about the model in a standardized format. The structure of the file is explained in detail [here](model_json.md). This information is used to compare, filter, and discover models, and to provide advanced, standardized information for all MHub models on the MHub.ai platform. Once your mdoel is accepted, you can find your model at [https://mhub.ai/models/my_model](https://mhub.ai/models/my_model).
