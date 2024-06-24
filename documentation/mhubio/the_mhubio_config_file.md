# MHub Configuration

This document describes the structure and use of the configuration files that are part of every MHub model and orchestrate the MHub pipeline in so-called workflows.

Each MHub model comes with a default workflow (`default.yml`) that is automatically executed when the container is started without additional arguments.

## Configuration File

The configuration file is a YAML file. It defines the order of the MHub-IO modules that are executed in the workflow and makes it possible to configure module parameters for all modules in a single file.

The configuration file has three sections.

### General Section

The first section is the `general` section. It is required for every configuration file and contains the `version` of the workflow, a `description` of the workflow and the `data_base_dir`. The `data_base_dir` is the base directory according to which all relative paths in the configuration file are resolved. The `description` is a human-readable description of the workflow. If you start the MHub container without specifying a workflow (e.g. by overwriting the default entry point) and in interactive mode (e.g. by executing `docker run -it --entrypoint mhub.run mhubai/pyradiomics`), a list of available workflows is displayed. You can select a workflow with the arrow keys and start it with the Enter key.

```yml
general:
  data_base_dir: /app/data
  version: 1.0
  description: run pyradiomics pipeline on dicom data
```

### Execute Chain

The second section is the `execute` section. It is required for every configuration file and contains a list of modules that are executed in the workflow in sequential order. MHub contains a variety of core modules to import, convert, filter and organize data. You can see here a [full list of all MHub core modules and their parameters](mhubio_modules.md). MHub core modules are available in the MHub base image and therefore shared across all MHub models. In addition to the core modules, custom modules can be added to the workflow. We automatically parse the `$model_name/utils` folder for custom modules and make them available in the `execute` chain of the workflow. A [detailed article how to write custom modules](how_to_write_an_mhubio_module.md) can be found here.

The following example describes a workflow that runs the following five MHub-IO core modules: `DicomImporter`, `DsegExtractor`, `NiftiConverter`, and `DataOrganizer`. The `PyRadiomicsRunner` in this example is a custom module, defined under `$model_name/utils/PyRadiomicsRunner.py`.

```yml
execute:
- DicomImporter
- DsegExtractor
- NiftiConverter
- PyRadiomicsRunner
- DataOrganizer
```

### Module Parameters

Most MHub-IO modules have configurable parameters that change the behaviour of the module and can be specified and overwritten in the `modules` section of the configuration file. You can see here a [full list of all MHub core modules and their configurable parameters](mhubio_modules.md).

```yml
modules:
  DicomImporter:
    source_dir: input_data
    import_dir: sorted_data
    sort_data: true
    merge: true
    meta: 
      mod: '%Modality'
      desc: '%SeriesDescription'

   DsegExtractor:
     roi: 
     - LIVER
     - LIVER+NEOPLASM_MALIGNANT_PRIMARY

  DataOrganizer:
    targets:
    - csv-->[i:sid]/pyradiomics.csv
    - nifit:mod=seg:origin=dicomseg-->[i:sid]/masks/[basename]
    - nifti-->[i:sid]/nifti/[d:mod]/[basename]
```

The `modules` section is used to define general parameters for each module. However, if you use a module more than once in a workflow, these configurations apply to each execution of the module. You can define the module parameters for a single execution directly in the `execute` section. Whatever is defined in the execute section takes precedence over the general module parameters.  

The order in which parameter values are prioritized is `execute > modules > default`.

```yml
execute:
- module: DicomImporter
  source_dir: input_data
  import_dir: sorted_data
  sort_data: true
  merge: true
  meta: 
    mod: '%Modality'
    desc: '%SeriesDescription'
```

### SegDB customization

Segmentation models can use segment IDs from our [SegDB](https://github.com/mhubai/segdb) to conveniently retrieve standardized ROI definitions.
The [DsegConverter](./mhubio_modules.md#dsegconverter) and the [RTStructConverter](./mhubio_modules.md#rtstructconverter) use the IDs specified in the `roi` metafield of the data (or any other metafield specified in `segment_id_meta_key` for the above modules, `roi` is the default) to retrieve the corresponding ROI definitions from the SegDB.

For example, an nrrd segmentation file labeled 0, 1, 2, and 3, where zero is the background and 1, 2, and 3 are the heart, left lung, and right lung, respectively, could contain the following MHub metadata: `nrrd:roi=HEART,LEFT_LUNG,RIGHT_LUNG` where `HEART`, `LEFT_LUNG`, and `RIGHT_LUNG` are the IDs of the corresponding ROIs in the SegDB.

You can overwrite the default SegDB entries and create your own SegDB entries if they are missing in the configuration file.

```yml
segdb:
  triplets:
    MY_CUSTOM_TRIPLET:
      code: 123
      meaning: custom code
  segmentations:
    MY_CUSTOM_SEGMENTATION:
      name: custom segmentation
      category: C_BODY_STRUCTURE
      type: MY_CUSTOM_TRIPLET
      color: [255, 255, 255]
    LEFT_LUNG:
      name: custom label and color for lung
      category: C_BODY_STRUCTURE
      type: T_LUNG
      modifier: M_LEFT
      color: [255, 0, 0]
```
