# MHub Documentation

MHub is a platform for medical AI models. It provides a standardized interface for AI models and a standardized way to run them. This document describes the basic principles of MHub and how to use it.

The following documentation is primarily intended for developers and contains information on how to customize configurations and describes the requirements and principles that must be met in order to contribute models to Mhub, but also for interested readers who want to learn more about the inner workings of Mhub.

## MHub Model Introduction

MHub models are bundled in a specific format. In addition, we require some specific metadata to be provided and set some rules on how source code and third-party resources can be provided with MHub models.

### Running MHub Models

All MHub models are shipped as Docker images. This article explains how you can run our MHub containers.

[documentation/mhub/run_mhub.md](documentation/mhub/run_mhub.md)

### Versioning in MHub

This document describes how MHub models are versioned to achieve a high degree of reproducibility.
Versioning plays an important role in our efforts to provide reproducible AI models, and together with our standardized input-output interface, makes it so easy to integrate MHub models into third-party systems.

[documentation/mhub/versioning.md](documentation/mhub/versioning.md)

### MHub Model Folder Format

Each MHub model is organized in a specific folder structure within our  [Models Repository](https://github.com/MHubAI/models/). This document describes the structure of the folder and how to set it up.

[documentation/mhub_models/model_folder_structure.md](documentation/mhub_models/model_folder_structure.md)

### Submitting a Model to MHub

If you plan to submit a model to MHub, we will gladly accept your submission. To ensure the high standard we apply to all models in MHub, we need to fulfill some requirements before we can accept your submission. This document describes the submission process and requirements for models submitted to MHub. Please read this document carefully to ensure a smooth and quick submission process and to minimize the effort on your and our side.

[documentation/mhub_contribution/contributing_a_model.md](documentation/mhub_contribution/contributing_a_model.md)

### Mhub Model Metadata

MHub models are not only about the implementation of the AI pipeline, which contains the environment setup and source code of the model, but also metadata describing the intended use of the model, inputs and outputs, training and evaluation data, and metrics. All of this information can be found on our [website](https://mhub.ai) on the model page. All of this meta information is contained in a single `meta.json` file.
The following document describes the structure of the file.

[documentation/mhub_models/model_json.md](documentation/mhub_models/model_json.md)

## MHub-IO

The MHub-IO framework provides several modules for importing, converting, and organizing medical files. Since the input and output of ML models can differ greatly in both file system level storage structure and file format (DICOM, NIFTI, NRRD) for the same modality (e.g., CT scan as input, segmentation as output), the modules in this framework allow easy adaptation to a well-defined and standardized I/O framework across multiple models. A key concept is that the model pipeline itself does not need to be changed. Instead, conversion and organization modules are used to provide data in the format used to create the model pipeline. Dealing with multiple patients, files, formats, and intermediate steps can quickly overload well-structured code with file system operations, making the code difficult for external readers to understand. MHub-IO therefore keeps track of file management internally. The MHub-IO pipeline is a sequential flow of modules, each of which retrieves and forwards data. Files are then requested not by their relative or absolute path, but by modality and format.

### Overview of MHub-IO Modules

The following document gives an overview and examples of all core modules provided with all MHub models.

[documentation/mhubio/mhubio_modules.md](documentation/mhubio/mhubio_modules.md)

### Writing MHub-IO Modules

This document describes how to write custom modules for MHub-IO, e.g., to provide your Runner modules to run a model pipeline or to run a custom conversion or data processing task.

[documentation/mhubio/how_to_write_an_mhubio_module.md](documentation/mhubio/how_to_write_an_mhubio_module.md)
