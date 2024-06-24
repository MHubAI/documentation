# MHubIO Modules

This document provides an overview of all the modules provided by my MHub-IO.
As mentioned earlier, MHub-IO is a modular framework. This means that all operations, e.g., converting data from one format to another, organizing files, resampling data, running an AI pipeline, etc., are divided into MHub- IO modules and executed sequentially at runtime. Although the tasks that the different modules perform on the data can be arbitrary, all modules share a similar data interface for incoming and generated data and can be configured through a single configuration file. The configuration file not only specifies all customizable parameters for all modules in a central location, but also describes the entire module chain. This makes the MHub-IO pipeline understandable and, due to the high flexibility of MHub- IO, easy to customize.

So far, we have six different types of modules for importing, filtering, converting, exporting, organizing, processing data and one for running AI pipelines. Below we'll discuss each module in detail.

## Importing data

`mhubio.modules.importer`

Every MHub-IO workflow starts with an importer module. Importer modules are responsible for reading input data provided to the MHub container. The importer module creates an internal semantic file structure that can then be used by subsequent modules to retrieve data through [semantic data type queries (DTQ)](./semantic_data_queries.md).

A term that will come up again and again in this and other documents is 'instance'. In MHub-IO, we call an instance a single case to be processed. For example, if we want to delineate heart masks based on a single CT image and give MHub-IO two such CT images, it'll treat each CT image as an instance.

### DicomImporter

`from mhubio.modules.importer.DicomImporter import DicomImporter`

As the name implies, a DicomImporter is used for Dicom data. Dicom data stores the 3D image of a patient in several 2D files (slices), which can be grouped in folders, e.g. per patient or per scan, but don't have to be, because dicom files are distinguishable by their header fields. Therefore, you can have your dicom data in a well-organized, nested folder structure or in a single folder full of files. Our dicom importer module can handle both. Internally, we use a tool called [dicomsort](https://github.com/pieper/dicomsort) to sort dicom files into a folder structure that separates all images (e.g. a 3D scan CT ) into separate folders.

Suppose you have Dicom data in unstructured files in a folder called `dicom` under `/local/path/to/data/`. The folder structure looks like this:

```text
/local/path/to/data/
├─ dicom
│  ├─ 000000.dcm
│  ├─ 000001.dcm
│  ├─ ...
```

Since MHub-IO runs inside a Docker container, you'll mount the data into the container. More on this (\tbd). Normally, your input data should be mounted in `/app/data/input_data`, e.g. `-v /local/path/to/data/dicom:/app/data/input_data/`. Internally, the structure looks like this:

```text
/app
├─ data
│  ├─ input_data
│  │  ├─ 000000.dcm
│  │  ├─ 000001.dcm
│  │  ├─ ...
```

This is the standard case and hence following with the default configuration of the dicom importer module. A minimal example to import data using this module hence is as follows:

```python
from mhubio.core import Config
from mhubio.modules.importer.DicomImporter import DicomImporter

config = Config()
Dicomimporter(config).execute()
```

Let's look at the configuration file and explain each configuration. Again, this is the default configuration, so there is no need to set the values. However, since MHub is still under development, it's safer to set all the values explicitly, since the default values can change.

```yaml
general:
  data_base_dir: /app/data

modules:
  DicomImporter:
    source_dir: input_data
    import_dir: sorted_data
    sort_data: true
      meta: 
        mod: ct
```

The `data_base_dir` property in the general section, which defines the root directory to which all data operations refer, is set to `/app/data`. This setting should remain the same in all MHub containers and therefore shouldn't be changed.

The source directory is set to `input_data`, which is the default value. The `import_dir` property sets the directory where the sorted data will be stored. The default value is `sorted_data`. Setting the `sort_data` flag ensures that Dicomsort is run with the input data. The `meta` property is used to specify the metadata of the imported data. The metadata is a dictionary that can store any information about the data. In the example, we can set the modality of our data to "CT" with the 'mod' key to indicate that we're using CT data. By default, the modality is set to `%Modality`, which is a placeholder (starting with `%` followed by a dicom header name) that is resolved depending on the imported file.

```yaml
general:
  data_base_dir: /app/data

modules:
  DicomImporter:
    source_dir: input_data
    import_dir: sorted_data
    sort_data: true
    match: true
    meta: 
      mod: '%Modality'
```

You should leave the source_dir and import_dir properties unchanged when using MHub-IO in an MHub (Docker) container.

Assuming you already have your Dicom data in a structured format, so there is no need to run dicomsort, MHub-IO supports two cases: Either you provide a folder containing only `*.dcm` files and make sure they belong to a single instance. The second scenario is that you provide multiple folders, each folder containing only `*.dcm` files and each representing a single instance.
In either case, running Dicomsort isn't a problem, as the tool will recursively go through the entire input structure, identify all the Dicom files, and then sort them into the specified folder. However, if you can ensure that your data is already structured, it makes sense to omit the sorting step for performance reasons, especially if the number of Dicom files is large. To omit the sorting step, set the `sort_data: false` flag in the config to false.

With the `meta` property you can specify the metadata that will be set to every imported dicom data. To see a list of the meta convention look (\tbd).

The DicomImporter automatically matches related DICOMSEG and RTSTRUCT dicom files with their images and imports these into a single instance. To prevent this automatic matching and to import every dicom instance into a separate mhub instance, set `match: false`. This can slightly increase the performance if e.g., segmentation data is present in your input folder but not relevant to the workflow you are running.

### NrrdImporter

`from mhubio.modules.importer.NrrdImporter import NrrdImporter`

The NrrdImporter is a special importer to import a single nrrd image file. We use this importer for our 3D Slicer extension workflows, but its use is generally discouraged. We may provide a more general-purpose nrrd and nifti importer in the future, and for now refer to the highly flexible FileStructureImporter.

### FileStructureImporter

`from mhubio.modules.importer.FileStructureImporter import FileStructureImporter`

The filestructure impoerter is a highly customizable importer that allows for complex imports like multiple input files per instance. To use the filestructure importer, your data must be consistently structured, e.g. your file structure should follow the same concept for all instances.

The file structure importer imports data by scanning the provided input data recursively and applying patterns that define how to import instances, bundles, files and metadata.

Consider the following example structure:

```text
input_data/
├─ random
├─ subjectA
│  ├─ dicom
│  │  ├─ 000000.dcm
│  │  ├─ 000001.dcm
│  │  ├─ ...
│  ├─ nifti
│  │  ├─ mask.nii.gz
├─ subjectB
│  ├─ dicom
│  │  ├─ 000000.dcm
│  │  ├─ ...
│  ├─ nifti
│  │  ├─ mask.nii.gz
...
```

You want to import dicom data and nifti data (multiple imports) for each subject (SubjectA, SubjectB, ...) hence each subject corresponds with an instance.

You then create the follwoing pattern in the config:

```yaml
FileStructureImporter:
  structures:
    - $patientID@instance/dicom@dicom:mod=ct
    - $patientID/nifti/mask.nii.gz@nifti:mod=seg
  excludes:
    - random/
  import_id: patientID
```

Under the `structures` attribute, you specify patterns that correspond to the path structure and contain instructions about what information to extract along the path.
You can also use filters and wildcards instead of folder names. A filter is a string like `/path` that matches only a folder named `/path/...`, but not `/dir/...`. A wildcard starts with a `$` and matches any folder name and populates it as a meta value under the key with the name of the wildcard. So `'/$path'` would match `'/path/...'` and `'/dir/...'` and create an instance attribute and metafield for all instances and data that will be imported subsequently, where `path=path` in the first example and `path=dir` in the second.

To create instances and import data, use the `@` annotation. To indicate that a folder is the root of an instance (i.e. all the data it contains belongs to a single instance), add `@instance` to the filter or placeholder of that folder. Instance imports can only occur once within a structure and cannot be nested across structures. To import data, instead of instance you write the string representation of the data type (dicom, nrrd, nifti, ...) and a list of meta `key=value` pairs separated by colons: `@datatype:meta1=val1:meta2=val2:...`.

You can specify an `import_id` which uniquely identifies the instances. The import id defaults to `_instance_id`, a unique, random uuid generated for each instance import. However, if you have a more complicated structure, your import_id could look like `PatientID/StudyID`. In the import_id field you can use any placeholder you have defined in your structures and separate it with `/`.

To exclude paths, you can specify a list of paths that shouldn't be scanned during the import in the `excludes` field. In the example above, we exclude the sub-folder `input_data/random` as it does not contain any relevant data.

The example in the configuration above would import each top-level folder as an instance and set the paientID attribute to the folder name. If there is a dicom folder within that instance folder, it'll be imported as dicom data, with the patientID metadata corresponding to the instance and an additional mod field set to ct. If there is a nifti folder within the instance folder that contains a file named mask.nii.gz, this file is imported as the nifti file for this instance, with the patientID and mod metadata set to seg according to the secend structure.

*Note: The file structure importer creates folders internally for each instance, but references imported files from the input_data directory. Newly created folders are placed in the created instacne directory.*

#### Import a single file

You can also use the file structure importer to import a single file.

```text
input_data/
├─ image.nii.gz
```

```yaml
FileStructureImporter:
  input_dir: 'input_data'
  structures: 
  - 'image.nii.gz@instance@nifti:mod=ct'
```

## Filtering Instances

`mhubio.modules.filter`

Filter modules are used to reduce the number of imported instances based on available data or attributes. In general, we advise using data importers carefully to avoid importing unwanted instances rather than importing them first and then filtering them.

### AttributeFilter

This filter can be used to filter instances by their attributes. In the configuration under `instance_attributes`, specify a dictionary with the attributes and the required values (or use an asterisk to indicate that only the key with any value must be present). The filter will then only let through instances that meet these criteria.

For example, you can use the following configuration to pass only instances with a specific SeriesInstanceUID:

```yaml
AttributeFilter:
  instance_attributes: 
    sid: '1.2.840.113654.2.55.49161834859837936399072171198980996986'
```

Note that the keys and values of the metadata depend on how the data was imported. So you have to make sure that the metadata you want to filter for is present in the instance metadata under the correct key.

## FileFilter

The FileFilter module filters instances according to the availability of certain files. You can specify a list of [DTQ](./semantic_data_queries.md) to check for the presence of files. All instances that don't have at least one matching file for all DTQs listed under `required` will be discarded.

```yaml
FileFilter:
  required:
    - nifti:mod=ct
    - nifti:mod=seg
```

## Convert Data

`mhubio.modules.converter`

Conversion modules bring data from one data type to another without changing the content of the data.

MHub-IO converters are designed to put data into a specific file format. So instead of having an importer that goes from `A -> X`, `B -> X`, `A -> Y`, `B -> Y`, etc., we have importers that put each supported file format into a specific format: `[A, B]-> X`.
When considering whether you need an importer, start from your AI Pipeline Runner module. What data format does it expect? For example, if your pipeline expects nifti files, then you use a NiftiConverter beforehand, which converts the existing data into nifti files.

### NiftiConverter

`from mhubio.modules.converter.NiftiConverter import NiftiConverter`

This module converts data into the nifti format from various formats (e.g. dicom or nrrd).

```yaml
NiftiConverter:
  in_datas: dicom|nrrd|mha:mod=ct|mr
  engine: plastimatch
  bundle_name: nifti
  converted_file_name: '[filename].nii.gz'
  allow_multi_input: false
  overwrite_existing_file: false
```

You can specify the source files by using the `in_datas` attribute and specifying a [semantic data type query (DTQ)](./semantic_data_queries.md) to fetch the data to be converted. The default is `dicom|nrrd|mha:mod=ct|mr`, which means that all dicom, nrrd, or mha data with modality ct or mr will be converted.

You can also specify the `engine` that will be used to convert the files. You can choose between plastimatch and dcm2niix. This only affects Dicom files, as we always use plastimatch as the conversion engine for NNRD files.

If you want to convert multiple files, set the flag `allow_multi_input`. Otherwise we'll give a warning if more than one file matches the DTQ specified under `in_datas` and convert only the first file.

If we find that a file already exists, we don't overwrite that file, but stop the conversion process for that file. We don't import the file into the internal file structure because we cannot verify its origin. If you want to overwrite existing files, set the flag `override_existing`.

The converted files will be placed in a bundle with the name specified under `bundle_name`. The default is `nifti`.

The converted files will be named according to the pattern specified under `converted_file_name`. The default is `[filename].nii.gz`, which means that the converted files will have the same name as the original files, but with the extension `.nii.gz`. You can use placeholders to specify the name of the original file. The following placeholders are available: [basename], [filename], [filext].

If you convert only a single file (e.g. a Dicom image to Nifti), you can set `converted_file_name` to a fixed filename like `image.nii.gz`. However, you shouldn't worry too much about filenames and usually keep this setting. If you need a specific filename for your output, you can use the DataOrganizer module to rename the files after conversion.

### MhaConverter

`from mhubio.modules.converter.MhaConverter import MhaConverter`

The *MhaConverter* module is a simple module to convert images in dicom, nrrd or nifti format into the mha format. The module behaves similar to the *NiftiConverter* described earlier.

We support two different conversion backends, Plastimatch and the [Panimg](https://pypi.org/project/panimg/) conversion library. You can specify which bakend to use via the `engine: plastimatch|panimg` attribute. The default is `plastimatch`.

```yaml
MhaConverter:
  in_datas: dicom|nrrd|nifti
  engine: plastimatch
  bundle_name: mha
  converted_file_name: '[filename].mha'
  allow_multi_input: false
  overwrite_existing_file: false
```

### TiffConverter

`from mhubio.modules.converter.TiffConverter import TiffConverter`

The *TiffConverter* module is a simple module to generate tiff files from dicom images.
The module behaves similar to the *NiftiConverter* described earlier.

***NOTE**: The TiffConverter module is currently only used for the conversion of dicom images to tiff files. We plan to support other data types in the future.*

```yaml
TiffConverter:
  in_datas: dicom:mod=sm
  bundle_name: tiff
  converted_file_name: '[filename].tiff'
  allow_multi_input: false
  overwrite_existing_file: false
```

### PngConverter

The PNG converter can be used to convert dicom data into an image format. This is useful for models strting from a image input. The `PngConverter` converter module takes a two dimensional dicom input and converts it into an image in png format.

```yml
PngConverter:
  in_datas: dicom:mod=cr|dx
  engine: itk
  allow_multi_input: flase
  bundle_name: png
  converted_dile_name: '[filename].png'
  overwrite_existing_file: false
```

The converter behaves similar to the other converters whith the above default settings. The size of the image is automatically determined depending on the size and resolution of the input image. If required, a fixed width can optionally be specified in the configuration with the `new_width` parameter. If set, the height is automatically adjusted to preserve the screen dimensions.

**Note, that the png converter accepts 2D images only as input. For dicom images that means ther must not be more than a single dicom slice available.**

**Note, that the png converter is yet in a beta state and will likely be improved and changes with upcoming versions of mhubio. For now, the png converter can be used to convert dicom images only but will be updated for nrrd, nifti and mha formats soon.**

### DsegConverter

`from mhubio.modules.converter.DsegConverter import DsegConverter`

The dicomseg converter module converts segmentations from nifti or nrrd files to a dicomseg (ref) file, which is a standard for bundling segmentations into a single file with standardized metadata such as colors and naming conventions.

To generate the dicomseg, we use the *itkimage2segimage* engine. You need a special configuration file that defines metadata for each input file and each segment within an input file. You can create such a configuration file in json format using their [web editor](http://qiicr.org/dcmqi/#/seg). In the dicomseg converter module you can specify the path to the configuration file with the attribute `json_config_path`. Note that we sort all the files we retrieve from the data types in `source_segs` in alphabetical order, so your json configuration file must look like this. In addition to the segmentation files, a Dicom series is needed to create an aligned Dicomseg file. Therefore, you must specify a data type pattern in the `target_dicom` field to select this dicom file. The default value is `dicom:mod=ct`, so all Dicom data with the modality ct will be selected. Note that you specify a single value here instead of a list, because there can only ever be a single Dicom file and the [DTQ](./semantic_data_queries.md) should uniquely identify that data.

```yaml
DsegConverter:
  model_name: Example Model
  target_dicom: dicom:mod=ct
  source_segs: nifti:mod=seg
  skip_empty_slices: True
  json_config_path: /app/models/example/config/dseg.json
```

To standardize all output, we provide an [internal database of segments](https://github.com/MHubAI/SegDB). Since all the segments you want to convert are in our database, you don't need to create and deploy a json configuration file. Instead, we can create it automatically based on the metadata field `roi`. The value must then be the ID of any segmentation from our database, e.g. `HEART`. The metadata field can be changed from roi to any meta key using the `segment_id_meta_key`, which defaults to `roi`.

```yaml
DsegConverter:
  model_name: Example Model
  target_dicom: dicom:mod=ct
  source_segs: nifti|nrrd|mha:mod=seg:roi=*
  skip_empty_slices: True
  segment_id_meta_key: roi
  body_part_examined: WHOLEBODY
```

We strongly recommend that you choose the latter method. You set the metadata of your rois when they're created (or imported), for example in your Runner module.

### RTStructConverter

`from mhubio.modules.converter.RTStructConverter import RTStructConverter`

The RTStructConverter converts segmentations from NIFTI, NRRD or MHA into RTStruct. This is useful if you want to use the segmentation e.g. in a treatment planning system (TPS) that does not support DICOMSEG.

```yaml
RTStructConverter:
  model_name: Example Model
  target_dicom: dicom:mod=ct
  source_segs: nifti|nrrd|mha:mod=seg:roi=*
  skip_empty_slices: True
  converted_file_name: seg.dcm
  bundle_name: null
  segment_id_meta_key: roi
  body_part_examined: WHOLEBODY
  use_pin_hole: False
  approximate_contours: True
```

Similar to the DSegConverter, the `target_dicom` specifies the source DICOM which the RT-Struct will be mapped to. The `source_segs` specifies the source segmentation file or files which will be converted to RT-Struct. When `skip_empty_slices` is enabled, empty segmentations will be ignored. The `converted_file_name` is used to set the file name used internally and the `bundle_name` specifies weather to create a bundle internally (you certainly don't need to touch these parameters; to change the file name of the RT structure exported from MHub, use the DataOrganizer module instead). The `segment_id_meta_key` works exactly as explained under the [DSegConverter](./mhubio_modules.md#dsegconverter) module above. The `use_pin_hole` flag specifies whether the RT-Struct should be created using a pin hole algorithm. If set to true, lines will be erased through your mask such that each separate region within your image can be encapsulated via a single contour instead of contours nested within one another. Use this if your RT Struct viewer of choice does not support nested contours / contours with holes. The `approximate_contours` flag specifies defines whether or not approximations are made when extracting contours from the input mask. Setting this to false will lead to much larger contour data within your RT Struct so only use this if as much precision as possible is required.

## Process Data

`mhubio.modules.processor`

Processing modules are modules that modify data without changing the data type or extract data and information from data.


### DsegExtractor

`mhubio.modules.processor.DsegExtractor`

The DsegExtractor module can be used to extract segmentations from a DICOMSEG file. Each segmentation will be exported as a separate NIFTI file by the module. The segmentations are automatically z-aligned to the target DICOM image which can be specified with the `target_dicom` parameter. The `in_datas` parameter then specifies the [semantic data type query (DTQ)](./semantic_data_queries.md) for the DICOMSEG file or files to be processed. Note, that although multiple segmentation files can be processed, they all must share the same target image. If you use `DicomImporter` with the `merge: true` option, this will be automatically resolved. The `roi` parameter is an optional value that can be used to specify the segmentations as a list of SegDB IDs. THe DsegExtractor will automatically extract the correct SegDB ID from the metadata if the DICOMSEG file was generated with MHub (i.e., by the DsegConverter) or if the metadata matches with a SegDB structure via it's code definition. If not, the roi metadata field on each generated NIFTI file will be set to the segment description. You can instead manually specify the SegDB ID with the `roi` parameter.

```yaml
DsegExtractor:
  target_dicom: dicom:mod=ct|mr
  in_datas: dicomseg:mod=seg
  bundle: 'nifti'
  roi: []
```

### RTStructExtractor

`mhubio.modules.processor.RTStructExtractor`

The `RTStructExtractor` works analogously to the `DsegExtractor` but for RTStruct files. The module extracts the segmentations from the RTStruct file and exports them as NIFTI files. The `target_dicom` parameter specifies the target DICOM image to which the segmentations will be aligned. The `in_datas` parameter specifies the [semantic data type query (DTQ)](./semantic_data_queries.md) for the RTStruct file or files to be processed. The `roi` parameter is an optional value that can be used to specify the segmentations as a list of SegDB IDs. The RTStructExtractor will automatically extract the correct SegDB ID from the metadata if the RTStruct file was generated with MHub (i.e., by the RTStructConverter) or if the metadata matches with a SegDB structure via it's segment name. If not, the roi metadata field on each generated NIFTI file will be set to the segment name. You can instead manually specify the SegDB ID with the `roi` parameter.

```yaml
RTStructExtractor:
  target_dicom: dicom:mod=ct|mr
  in_datas: rtstruct:mod=seg
  bundle: 'nifti'
  roi: []
```

## Run AI Pipelines

`mhubio.modules.runner`

Runner modules are generative modules that run an AI pipeline and produce new data.

### NNUnetRunner

`from mhubio.modules.runner.NNUnetRunner import NNUnetRunner`

Due to the great success of the NNUnet architecture, we decided to provide a runner that runs the NNUnet pipeline and provides a standard MHub implementation. You'll use this runner for all NNUnet tasks. Make sure you specify the weights in your Docker image, as seen in [this example](https://github.com/MHubAI/models/blob/main/models/nnunet_liver/dockerfiles/Dockerfile).

```yaml
NNUnetRunner:
  folds: all
  nnunet_task: Task400_OPEN_HEART_1FOLD
  nnunet_model: 3d_lowres
  roi: HEART
```

## Export Information

`mhubio.modules.exporter`

Exporter modules are used to export raw, derived or aggregated information on the data collected internally during the execution of the MHub workflow.

### JsonSegExporter

The JsonSegExporter exports files and their SegDB segmentation IDs. You should use the same file targets in the JsonSegExporter as in the DataOrganizer to export information about the SegDB IDs of the files that match the files.

```yaml
  JsonSegExporter:
    targets:
    - nifti:mod=seg-->[i:sid]/[basename]

  DataOrganizer:
    targets:
    - nifti:mod=seg-->[i:sid]/[basename]
    - json-->segdef.json
```

The exporter will generate a json file of the following structure, with one entry for each file matching any of the set targets.

```js
[
  {
    "file": "1.2.826.0.1.3680043.8.498.99748665631895691356693177610672446391/image.nii.gz",
    "labels": {
      0: "HEART",
      // ...
    }
  },
  // ...
]
```

### ReportExporter

The ReportExporter module allows you to collect information from the MHub pipeline and export it to a custom report. The report is exported as a json file with the metadata `json:mod=report`.

To specify the data to include in your report, you write directives in the `includes` attribute, each of which creates a label and value that will be added to the report. There are four different categories of directives, described in detail below: static text, instance attributes, instance files, and instance data.

The `format` attribute allows you to specify the format of the report. There are three different formats: `separated`, `nested`, `compact`. In *separated* format the json report is an array of objects, each containing a label and a value. In the *compact* format, the json report is an object with the labels as keys and the values as values. The *nested* format creates a nested array based on the labels, where you can separate the levels with a `/`.

```yaml
ReportExporter:
    format: compact
    includes: []
```

#### Static

You can include static values with a `static` directive. Specify a `label` and `value` for the static value as in the following example.

```yaml
ReportExporter:
  includes:
  - static:
    label: Model
    value: Example Model
```

#### Attributes

You can insert any attribute of your instance. Instance attributes are set, for example, as they are defined in your importer module.

To include an attribute, use the key `attr` and specify the name of the attribute as `value`. You can also specify a `label` to set the name of the attribute in the report.

In the following example, you include the attribute `sid` (if present in the instance) as a value and specify `SeriesInstanceUID` as the label.

```yaml
ReportExporter:
  includes:
  - attr: sid
    label: SeriesInstanceUID
```

The generated report looks like this:

```json
[
  {
    "label": "SeriesInstanceUID",
    "value": "1.2.826.0.1.3680043.8.498.3455481963121056024207396004934459758"
  }
]
```

Or with `format` set to `compact`:

```json
{
  "SeriesInstanceUID": "1.2.826.0.1.3680043.8.498.3455481963121056024207396004934459758"
}
```

#### Files

You can also add information about files contained in an instance, such as imported, converted, or generated files. To add a file directive, start with the keyword `files`. This will include all files that are available in the instance in the directive. You can also specify a [DTQ](./semantic_data_queries.md) to include only files that match the DTQ as the value for the `files` keyword.

You need to specify with the `aggregate` attribute how the value will be derived from the filtered files. If you select `count`, the reported value is the number of all filtered files. If you select `list`, the value will be a list of all files. In the `pattern` attribute you can specify how the individual files should be displayed (all placeholders described in *DataOrganizer* are available). You can also convert the list to a string by specifying the `delimiter` attribute. The files will then be joined into a single string by the delimiter.

```yaml
ReportExporter:
  includes:
  - files: 
    label: total number of files
    aggregate: count
  - files: any:mod=ct
    label: ct files
    pattern: '[i:id]/[basename]'
    aggregate: list
  - files: 
    label: list of modalities
    pattern: '[d:mod]'
    aggregate: list
    delimiter: ','
```

#### Data

The most important include directive is the data directive, which allows all derived data (e.g., the RunnerOutput score generated by a prediction model) to be included in the generated report.

There are two different types of outputs: Value outputs and Class outputs. Value outputs have a single value, such as a model's prediction. Class outputs have multiple classes, each with a prediction score. The value of a class output is the selected class (the selection of the class depends on the pipeline, but is usually the class with the highest prediction score).

Use the `data` directive for a data directive and specify the name (or a query string) as the value. A query string is similarily to query string used for file queries, but instead of a data type (e.g., dicom or nrrd) you specify the name of the data field. You can read more about semantic data queries [here](./semantic_data_queries.md). Specify the value you want to display under the `value` keyword. For value and class outputs you can specify `value: description`, `value: label` or `value: value` to report the description, label or value of the output data. For value outputs you can additionally specify `value: type` to report the data type (e.g. string, integer, float) of the output. For class outputs you can specify insead the attribute `class: $classname` and use the class name to report insead the description, label, or value of that class.

```yaml
ReportExporter:
  includes:
  # value output (name: ags)
  - data: ags
    label: Agatston Score Description
    value: description
  - data: ags
    label: Predicted Agatston Score
    value: value
  # class output (name: rs)
  - data: rs
    label: Risk Group Description
    value: description
  - data: rs:metakey=value
    label: Risk Group probability (low)
    class: low
    value: probability
  - data: rs AND any:metakey=value
    label: Predicted Risk Group
    value: value
```

The name of a value output or class output is as defined in the ModelRunner module using the MHub-IO `@ValueOutput.Name` and `@ClassOutput.Name` decorators. The name of a class for a class output is as defined by the first argument to the `@ClassOutput.Class` decorator.

If the `data: <query>` query fetches more than one matching data instance you must use an aggregate function. By default, `aggregate: one` is set to `one`, which will throw an error if not exactly one file is returned. The following aggregate functions are supported:

- **one**    (returns the first matching value, fails if more than one matching value is found)
- **first**  (returns the first matching value)
- **list**   (returns a list of all matching values, a `delimiter: ','` for concatenation can optionally be specified)
- **count**  (returns the counted number of matching values)
- **sum**    (requires numeric values, returns the sum of all matching values)
- **avg**    (requires numeric values, returns the average of all matching values)
- **min**    (requires numeric values, returns the minimum of all matching values)
- **max**    (requires numeric values, returns the maximum of all matching values)

## Organize Data

`mhubio.modules.organizer`

Organizer modules are modules that don't change data or data types, but copy files and folders to a new, specified location and structure.

### FileRemover

This module removes all files that match a [DTQ](semantic_data_queries.md) in any instance.
This can be useful to reduce disk space for files created in the meantime that are no longer needed.

Specify the DTQ in the `query` attribute. If you want to remove all files except those that match the DTQ, you can invert the DTQ with the `NOT` statement, e.g. `NOT nifti:mod=seg`.

```yaml
FileRemover:
  query: nifti:mod=seg
```

### DataOrganizer

`from mhubio.modules.organizer.DataOrganizer import DataOrganizer`

The DataOrganizer module makes it possible to fetch data from the internal data structure and arrange it in a new structure using placeholders.
To organize your data, you can specify a list of directives in the 'target' attribute. A directive has the following structure: `<query>--><path_structure>`, where `<query>` is a data type descriptor that starts with a file type (nifti, nrrd, dicom, ...), followed by a colon-separated list of `key=value` pairs and a `<path_structure>` where the matching data should be stored. The path structure may contain wildcards that are resolved individually for each file that matches the query:

- `[random]` -> random uuid4 string
- `[path]` -> relative path of the datatype
- `[basename]` -> basename of the datatype
- `[filename]` -> filename of the datatype
- `[filext]` -> file extension of the datatype (or empty)
- `[i:id]` -> instance id
- `[i:...]` -> any attribute from the instance
- `[d:...]` -> any metadata from the datatype

Wildcards starting with `i:` are used to access any instance attribute, e.g., `[i:id]` is replaced with the ID of an instance. Wildcards starting with `d:` are used to access any meta field from the data, e.g. `[d:mod]` is replaced with the Modality of the datatype such as CT, MR, SEG, etc.

Normally, the data organizer contains only data that has been confirmed. MHub internally checks whether files and folders have actually been created, and assigns a confirmation flag if positive. Note that this doesn't include content checks. Data cannot be confirmed if an error occurs during creation after the data object has already been created in the internal structure. To disable this behaviour and include unconfirmed files, set the `require_data_confirmation` attribute to `false`, which defaults to `true`.

```yaml
DataOrganizer:
  target_dir: output_data
  require_data_confirmation: true
  targets:
  - nifti:mod=ct-->[i:sid]/image.nii.gz
  - nrrd:mod=seg-->[i:sid]/seg/[d:roi].nrrd
  - dicomseg-->[i:sid]/casust.dcm
```

In the above example, the targets resolve as follows:

- get all nifti image files that have the modality `ct`. Then put them in a folder with the name of the sid attribute from the instance of the data at `/app/data/output_data/<sid>` as `image.nii.gz`
- get all nrrd segmentation files that have the modality `seg`. Then put them in a folder with the name of the sid attribute from the instance of the data under `/app/data/output_data/<sid>/seg`. To resolve the filename, check the metadata of the data for the `roi` attribute and use it as the filename with the `.nrrd` extension attached.
- retrieve all dicom segmentation files. Then put them in a folder with the name of the sid attribute from the instance of the data under `/app/data/output_data/` as `casust.dcm`

In the first and last examples, a single file is clearly expected to be found by the [semantic data type query (DTQ)](./semantic_data_queries.md), since a static filename is used. If multiple files matched the DTQ `nifti:mod=ct`, only the last file would be returned, since it overrides the other files due to the same filename.

Note that in the above example, multiple instances are supported and each instance is organized in its own folder with the instance ID name. You can omit the `/[i:sid]/` part of the path structure if you know you're only running a single instance at a time to simplify your output structure.