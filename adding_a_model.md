## Introduction

In this guide, we'll go over the steps one needs to follow to add a new model to the [[MHub]] `models` repository.

To better document the process and provide a practical example, we are going to add the [LungMask](https://github.com/JoHof/lungmask) model to MHub.

## Prerequisites

Before we get started, make sure to keep the following in mind:
- The model you want to add must have a public repository, and the model weights must be publicly available. 
- The license of the model repository and the model weights must comply with the MHub requirements.

If this is not the case, but you are one of the developers and still want to add the model to MHub while waiting for the public release, you might want to consider adding it under `MHubAI/models-private`.

## Steps to Add a Model to MHub

### Step 0: Prepare your model

If you are one of the developers of the model you want to add to MHub, make sure the model is ready for public release before starting the process. This means that you should:

- Clean up the code: Make sure your code is well-documented, readable, and free from any unnecessary files or directories.
- To ensure all of the MHub integration work, prepare the pipeline to be as modular as possible. For instance, try not to have all of the conversion, preparation, and pre-processing steps in one big, messy `.py` file. MHub offers several glue layers and tools you can use for data conversion (e.g., DICOM to NIfTI or NRRD), preparation (e.g., handling several different DICOM series at once), and post-processing (e.g., converting the data back to a standardized format such as DICOM SEG). Therefore, focus on preparing a pipeline to pre-process and process data - MHub can help with the rest!

We will further expand on the importance of the modularity pre-requisite in the [MHub Integration](#mhub-integration-and-different-entry-points) section.

### Step 1: Prepare the branch

As a first step, make sure to create a branch in the `MHubAI/models` repository (or `MHubAI/models-private`). Since we are working on adding several models and functionalities, branching plays an important role in keeping everything tidy. Make sure you name your branch starting with the `m-` (for "model") prefix. In this example, we'll create a branch named `m-lungmask` in the `MHubAI/models` repository.

Once the branch is created and cloned on your locale, you can start by creating the directory structure. The structure should look like the following:

```
.
├── README.md
├── __init__.py
├── config
├── dockerfiles
├── scripts
└── utils
```

We will go over every step and file in detail in the following sections, but for the time being, it's helpful to know that:
* The `config` directory will store all of the `yaml` files used to define the workflow.
* The `dockerfile` directory will store all of the Dockerfiles used to build the container from scratch. This directory must be organized in subdirectories, each for every version of the CUDA libraries used by the model (e.g., `nocuda`, `cuda12.0`, and so on). To keep things as transparent as possible, we advise not to use `COPY` or `ADD` commands inside the Dockerfile (as it's easy to lose track of what file version was used in the building process and so on). Instead, we always advise cloning from git and installing libraries and packages specifying commits/versions, so that everything stays as reproducible as possible.
* The `scripts` directory will store the Python scripts used as entry points for the container. In most of the cases (if not all), the default entrypoint will be script named `run.py`, responsible for running the DICOM-to-DICOM workflow: from the conversion of DICOM Series into files that can be processed by the AI pipeline, through the processing, to the post-processing and conversion back to DICOM SEG. Other files that can be stored in the `scripts` directory are, for instance, the `slicer_run.py` entrypoint (for the integration with 3DSlicer), `xnat_run.py`, `kaapana_run.py`, and so on - depending on MHub integrations and how different platforms/tools expect the MHub container to run.
* Last but not least, the `utils` directory will store all of the custom MHub modules developed for the model we want to add. By default, we expect the directory to contain a `${MODEL_NAME}Runner.py` file, which stores the code to run the model in a MHub-compliant fashion (which is, populating all the fields MHub needs to ensure transparency and work as intended - such as information on how the data should be handled by the pipeline in input and output, what directories and files should the pipeline create in the container filesystem, and so on).

### Step 2: Start working on the model Dockerfile

As mentioned above, a model can have several Dockerfiles that users can use to build different Docker images with different CUDA versions. However, all of the models should share a base image - which also comes in different flavors, depending on the CUDA version the model needs/the user decides to go for. 

For instance, as of May 2023, we support three different base images (`mhubai/base`): a lighter one that comes without CUDA (`mhubai/base:nocuda`), and two that have CUDA 11.4 and 12.0 pre-installed (`mhubai/base:cuda11.4` and `mhubai/base:cuda12.0`, respectively). 

The base image comes with a number of Python libraries pre-installed, the MHub I/O framework, and a few other tools we need for conversion between file formats. The Dockerfiles used to generate the base images are available and can be inspected at [`MHubAI/models/base`](https://github.com/MHubAI/models/tree/main/base/dockerfiles). As part of the MHub effort, we maintain all of the (base and model) Dockerfiles public and all of the images on dockerhub at https://hub.docker.com/r/mhubai. 

---

Before writing the runner module, creating the config files and the entrypoint scripts, testing the model in the Docker container is advised to understand what components are missing and streamline the creation of the model-specific Dockerfile.

For our example, let's start from the `mhubai/base:cuda12.0` image. For context, the `sample_data` folder stores publicly available, anonymized sample scans from the 3DSlicer sample collection.

```
docker run \ 
	--volume=/home/dennis/Desktop/sample_data/input_nii:/app/data/input_data \
	--volume=/home/dennis/Desktop/sample_data/output_data:/app/data/output_data \ 
	--gpus all --rm -it --entrypoint bash mhubai/base:cuda12.0
```

Following the instructions available in the original repository, we can try to install `LungMask` running:

```
pip3 install --no-cache-dir git+https://github.com/JoHof/lungmask
```

Some of the requirements will already be satisfied, while some packages will be installed in a specific version. However, we will notice that none of the dependencies is broken or need extra work to set up. If manual intervention is needed, the model-specific Dockerfile should be customized as the user sees fit.

N.B. - All of the base images are equipped with Python 3.8.10 (end-of-life at the end of 2024). If the model requires a specific older version of Python to work, you might want to check if that breaks some functionalities in MHubIO. Upgrading should never be a problem, but downgrading severely (e.g., to Python 3.6 - way past its end-of-life) could cause the framework to malfunction. It's also essential to notice that, being the MHub containers self-contained, there should not be any security risks in running an obsolete version of Python - if needed and supported.

---

After all of the dependencies and pipeline libraries are installed, we can take care of downloading the weights. Most libraries (e.g., LungMask, PlatiPy, nnU-Net) will download the weights upon the first use of the pipeline (usually running a Python scripts based on `wget`, fetching the files from GitHub, Zenodo, Dropbox, and other sources). To make the execution of the container faster, we can avoid that by manually downloading the weights. The location of the weights can usually be found by looking in the repository for the code used to download the weights or sometimes by inspecting the output messages.

For instance, when testing LungMask in the container:

```
lungmask data/input_data/image.nii.gz data/output_data/lungmask_output.nii.gz --modelname R231
```

We realize pretty quickly where the authors are pulling the weights from and where they are storing them. We will usually need to pull the weights in the exact location to make sure the pipeline works as intended (although some pipeline allow the user to specify the location of the weights file).

```
> lungmask 2023-05-03 16:40:35 Load model
> lungmask 2023-05-03 16:40:35 Read input: data/input_data/image.nii.gz
> lungmask 2023-05-03 16:40:35 Infer lungmask

> Downloading: "https://github.com/JoHof/lungmask/releases/download/v0.0/unet_r231-d5d2fc3d.pth" to /root/.cache/torch/hub/checkpoints/unet_r231-d5d2fc3d.pth
```

N.B. - To make sure everything works as intended, we will also need to pay attention to the file format (e.g., in some cases, we'll need to `untar` archives) and the directory structure of the location where the weights are saved (e.g., nnU-Net, PlatiPy, TotalSegmentator, and others are all based on nnU-Net, but expect weight files in a different location, organized in a slightly different directory structure).

### Step 3: Writing the Config File

The configuration file should store all the steps of the workflow MHub will execute. If, for instance, we want to run the LungMask pipeline on DICOM Series directly (as in most cases in medical applications), we can use the MHub's module to import and convert such data in a format supported by LungMask (i.e., NIfTI files). In pseudo-code, we need something like the following:

```
IMPORT data from $DATA_DIR
SORT and check the data in $DATA_DIR
CONVERT the data to NIfTI format
PROCESS the NIfTI data with LungMask
CONVERT the data to DICOM SEG
```

Translating this in MHub's config file standard:

```
general:
	data_base_dir: /app/data

modules:
	UnsortedInstanceImporter:
		input_dir: input_data
	DataSorter:
		base_dir: /app/data/sorted
		structure: '%SeriesInstanceUID/dicom/%SOPInstanceUID.dcm'
	NiftiConverter:
		engine: plastimatch
	[...]
	DsegConverter:
		dicomseg_json_path: /app/models/lungmask/config/dseg.json
		skip_empty_slices: True
```

This will parse all the DICOM data in `/app/data/input_data` (mapped by the user when running the container), sort these data by separating different DICOM Series (following the specified `structure`), and at the end, convert the results back to DICOM SEG using the configuration file at `dicomseg_json_path`.

The only thing missing in the config file is the module used to run the LungMask pipeline, which should look like this:

```
	LungMaskRunner:
		extra_argument_1: blabla
		extra_argument_2: blabla
```

Note: to learn more about the base MHub components (such as `UnsortedInstanceImporter`, `DataSorter`, `NiftiConverter`, `DsegConverter`, and so on), please see [the pertinent documentation (WIP)](https://github.com/MHubAI/docs).

### Step 4: Writing the Runner module

As anticipated, each model needs a Runner module to be executed. This module contains a model-specific Class that inherits specific properties from the base `ModelRunner` class in `mhubio.modules.runner.ModelRunner`. This ensures all models can be essentially swapped while keeping the input/output framework standardized.

The Runner module is a very simple module that wraps the code needed to run the model into a Class whose goal is to "glue" how the model is invoked to the I/O structure. If the pipeline is run through a Python script, you can copy the relevant code in the Class. If, on the other hand, the pipeline comes with Python APIs or with a bash executable, you can invoke those in the Class.

LungMask comes with both Python APIs and bash-executable functions. For now, let's focus on wrapping the code we tested above (i.e., the bash-executable). The best practice in this case using `subprocess`. The chunk of code that will actually run the model is as simple as:

```
subprocess.run(["lungmask",
				"data/input_data/image.nii.gz",
				"data/output_data/lungmask_output.nii.gz",
				"--modelname",
				"R231"],
				check=True, text=True)
```

#### MHubIO

The snippet above will save the segmentation mask in the Docker container file system. At this stage, it would then be our responsibility to implement checks to ensure the downstream post-processing and conversion get appropriately executed. Furthermore, handling all the different files can become very hard whenever a pipeline includes several steps, especially if we want the pipelines to be exchangeable as anticipated.  For instance, imagine processing hundreds of subjects in one go with a pipeline that works in several steps (e.g., a coarse-to-fine segmentation pipeline running a first segmentation model, cropping a cube out of a CT scan around that segmentation mask, to run a second segmentation then).

Without anything linking files together, we would have to rely merely on file names - and repeat this operation for every new model we want to include in the collection, keeping track of all the naming conventions and adapting/extending them to the particular use case.

As the purpose of MHub is to abstract this effort from the user, we developed an Input/Output (I/O) framework, namely MHubIO, which helps with file management. Whenever a DICOM series is ingested by the `UnsortedInstanceImporter`, all the subsequent steps keep track of every piece of data generated, its location on the file system, its link to the original DICOM CT Series, and so on and so forth.

The object generated from this process is called `Instance`, and can be inspected and interrogated at any moment in time. MHub outputs a description of the instance at every step of the processing, which looks like the following:

```
Instance  [/app/data/sorted/1.2.826.0.1.3680043.8.498.99748665631895691356693177610672446391]

├── id: cba50389-8f96-475c-bbfd-19037a43bbbc
├── ref: 1.2.826.0.1.3680043.8.498.99748665631895691356693177610672446391
├── SeriesID: 1.2.826.0.1.3680043.8.498.99748665631895691356693177610672446391
├── DICOM [/app/data/sorted/1.2.826.0.1.3680043.8.498.99748665631895691356693177610672446391/dicom] ✓
│   ├── mod: ct
├── NIFTI [/app/data/sorted/1.2.826.0.1.3680043.8.498.99748665631895691356693177610672446391/image.nii.gz] ✓
│   ├── mod: ct
├── LOG [/app/data/sorted/1.2.826.0.1.3680043.8.498.99748665631895691356693177610672446391/_pypla.log] ✓
│   ├── mod: ct
│   ├── log-origin: plastimatch
│   ├── log-task: conversion
│   ├── log-caller: NiftiConverter.dicom2nifti
│   ├── log-instance: <I:/app/data/sorted/1.2.826.0.1.3680043.8.498.99748665631895691356693177610672446391>
```

Furthermore, if something goes wrong, the user will be notified of when and where this happened. For instance, a failed processing run could look like the following (notice the ✗ nearby the piece of data missing):

```
├── NIFTI [/app/data/sorted/1.2.826.0.1.3680043.8.498.99748665631895691356693177610672446391/roi1_lungs.nii.gz] ✗
│   ├── mod: seg
│   ├── roi: roi1_lungs
├── NIFTI [/app/data/sorted/1.2.826.0.1.3680043.8.498.99748665631895691356693177610672446391/roi2_lunglobes.nii.gz] ✗
│   ├── mod: seg
│   ├── roi: roi2_lunglobes
```

---

Back to the LungMaskRunner module, we can use MHubIO to abstract most of the file management tasks. The only thing we will need to do is use decorators to link all of the instance data together in the `class LungMaskRunner(ModelRunner).

For instance, in the LungMask example, we want to take the NIfTI file storing the converted CT Series and produce two ROIs/segmentation masks, one for the lungs and one for the lung lobes. This will translate into:

```
@IO.Instance()

@IO.Input('image', 'nifti:mod=ct', the='input ct scan')

@IO.Output('roi1_lungs', 'roi1_lungs.nii.gz', 'nifti:mod=seg:roi=roi1_lungs', the='predicted segmentation of the lungs')

@IO.Output('roi2_lunglobes', 'roi2_lunglobes.nii.gz', 'nifti:mod=seg:roi=roi2_lunglobes', the='predicted segmentation of the lung lobes')
```

Where the first two fields are fundamental, but the rest is free-text and should be used for documentation and transparency purposes.

After these objects are defined, we can access them and their properties as we would do with any other attribute. For instance, instead of specifying the path to the output NIfTI file, we can use `roi1_lungs.abspath` - and the I/O framework will handle the rest.

```
# bash command for the lung segmentation
bash_command = ["lungmask"]
bash_command += [image.abspath] # path to the input_file
bash_command += [roi1_lungs.abspath] # path to the output file
bash_command += ["--modelname", "R231"] # specify lung seg model
  
self.v("Running the lung segmentation.")
self.v(">> run lungmask (R231): ", " ".join(bash_command))

# run the lung segmentation model
bash_return = subprocess.run(bash_command, check=True, text=True)
```

[...]

---

Of course, using the Python APIs or writing the `task` method purely in Python would be very similar, as all of the `mhubio` components would not change (rather, the snippet to run the model would).

### Step 5: Writing the entrypoint script

The decorators from the IO module of  `mhubio.core` also need to be used to specify custom model-dependent arguments before putting them in the config file.

The following two are, therefore, both needed:

```
# as a decorator in the LungMaskRunner module
@IO.Config('batchsize', int, 64, the='Number of slices to be processed simultaneously. A smaller batch size requires less memory but may be slower.')
```

```
# in the config file config.yml

LungMaskRunner:
  batchsize: 64
```

[...]

### Step 6: Putting everything together

Following the steps above, and after making sure we know where to download from all of the model weights, the model-specific Dockerfile will look like the following:

```
# Specify the base image for the environment
FROM mhubai/base:cuda12.0
  
# Install LungMask
RUN pip3 install --no-cache-dir \
  git+https://github.com/JoHof/lungmask

[...]

# pull weights for the lung segmentation 2D U-Net model
ENV WEIGHTS_DIR="/root/.cache/torch/hub/checkpoints/"
ENV WEIGHTS_URL="https://github.com/JoHof/lungmask/releases/download/v0.0/unet_r231-d5d2fc3d.pth"

RUN wget --directory-prefix ${WEIGHTS_DIR} ${WEIGHTS_URL}

# pull weights for the lung lobes segmentation 2D U-Net model
ENV WEIGHTS_URL="https://github.com/JoHof/lungmask/releases/download/v0.0/unet_ltrclobes-3a07043d.pth"

RUN wget --directory-prefix ${WEIGHTS_DIR} ${WEIGHTS_URL}

# Default run script
CMD ["python3", "/app/mhub/lungmask/scripts/run.py"]
```

[...]

In general, it is best practice to run as few `RUN` instructions as possible to limit the number of intermediate layers created by the building process.

In this case, the Python script at `/app/mhub/lungmask/scripts/run.py` will use the Runner module and the YAML configuration file to run a DICOM to DICOM workflow.

---

```
docker build --tag mhubai/lungmask:cuda12.0 --no-cache .
```

```
docker run \
  --volume=/home/dennis/Desktop/sample_data/input_dcm:/app/data/input_data \
  --volume=/home/dennis/Desktop/sample_data/output_data:/app/data/output_data \
  --gpus all --rm -it --entrypoint bash mhubai/lungmask:cuda12.0
```

## MHub Integration and Different Entry Points

Depending on the application that is leveraging the MHub container, we might want to use different entry points.

