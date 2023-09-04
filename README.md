# MHub Documentation

The following documentation is primarily intended for developers and contains information on how to customize configurations and describes the requirements and principles that must be met in order to contribute models to Mhub, but also for interested readers who want to learn more about the inner workings of Mhub.

## MHub-IO

The MHub-IO framework provides several modules for importing, converting, and organizing medical files. Since the input and output of ML models can differ greatly in both file system level storage structure and file format (DICOM, NIFTI, NRRD) for the same modality (e.g., CT scan as input, segmentation as output), the modules in this framework allow easy adaptation to a well-defined and standardized I/O framework across multiple models. A key concept is that the model pipeline itself does not need to be changed. Instead, conversion and organization modules are used to provide data in the format used to create the model pipeline. Dealing with multiple patients, files, formats, and intermediate steps can quickly overload well-structured code with file system operations, making the code difficult for external readers to understand. Mhubio therefore keeps track of file management internally. The MHub-IO pipeline is a sequential flow of modules, each of which retrieves and forwards data. Files are then requested not by their relative or absolute path, but by modality and format.

### Overview of MHub-IO Modules

The following document gives an overview and examples of all core modules provided with all MHub models.

[documentation/mhubio/mhubio_modules.md](documentation/mhubio/mhubio_modules.md)

### How to Write an MHub-IO Module

This document describes how to write custom modules for MHub-IO, e.g., to provide your Runner modules to run a model pipeline or to run a custom conversion or data processing task.

[documentation/mhubio/how_to_write_an_mhubio_module.md](documentation/mhubio/how_to_write_an_mhubio_module.md)

## MHub Model

MHub models are bundled in a specific format. In addition, we require some specific metadata to be provided and set some rules on how source code and third-party resources can be provided with Mhub models.

## The MHub Model Folder Format

Each MHub model is organized in a specific folder structure within our  [Models Repository](https://github.com/MHubAI/models/). This document describes the structure of the folder and how to set it up.

[documentation/mhub_models/model_folder_structure.md](documentation/mhub_models/model_folder_structure.md)

## Mhub Model Meta Data

For each model, a `meta.json` file that describes the model in a defined format must be provided. The following document describes the structure of the file. 

[documentation/mhub_models/model_json.md](documentation/mhub_models/model_json.md)