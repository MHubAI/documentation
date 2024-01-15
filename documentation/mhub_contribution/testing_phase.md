# MHub Contribution - Testing Phase

If your model reaches the testing phase of the [contribution process](./contributing_a_model.md#submission-timeline), congratulations, you have passed the code review and are almost done.

To ensure that each MHub model works as expected by the contributor and generally produces reliable and consistent output, your model must go through our testing process, which is explained in detail below.

Once your model has reached `Ready for Testing` status, please provide us with the information listed below so that we can proceed with testing your model.

## Prerequirements

To enter the test phase, your contribution must meet certain requirements.

1. Your PR must be up to date with the `main` branch.
2. Your PR must pass all GitHub Actions checks.
3. All review comments have been processed and resolved.

## Test Data

To test your proposed MHub implementation, you must first find suitable test data with which to test it. The test data must fulfill some criteria to be suitable for testing.

1. The test data must be publicly accessible.
2. The test data must be licensed under an open source license.
3. The test data must generally be compatible with the model you are using on MHub.
4. The test data has not been used for training or tuning the model.
5. The test data must be available on [IDC](https://portal.imaging.datacommons.cancer.gov/) (if available, see note below).

You can browse a huge collection of publicly available images on [IDC](https://portal.imaging.datacommons.cancer.gov/), which offers great filtering and search capabilities and integrates seamlessly with MHub. We encourage you to use data from IDC whenever possible.

*Note: If you cannot find suitable test data on IDC, please contact us (e.g. you can open an issue on our GitHub model repository) and suggest an alternative data source. We will then decide whether the data is suitable for testing.*

## Test Procedure

Now that you've found some suitable data to test your implementation on, you can run through the following steps to test your model.

1. Prepare your submission

    The testing procedure takes a bit of time on your and our side so please make sure to test and submit the right code.

    1.1. Ensure that your PR is up to date with our `main` branch.  
    1.2. Commit all changes to your local repository.  
    1.3. Push all changes to your remote fork.

2. Get the latest MHub base image

    Ideally, you're base image is up to date during development. However, to ensure you have the latest base image, refetch it from Docker Hub.

    ```bash
    docker pull mhubai/base:latest
    ```

3. Build your model.

    To create your model, first navigate to your project folder where your Dockerfile is located. `$model_name` is the name of your model. Please replace *"my_model "* with the name of your model.

    ```bash
    export model_name="my_model"
    cd /path/to/your/projects/models/models/$model_name/dockerfiles
    ```

    When you build your model now, you need to reference the MHub implementation code. Since your model is not yet submitted, we cannot automatically pull your implementation from our models repository as we normally would. Therefore, you need to provide the URL of the form you used to create the PR (this is the repository where your model resides until the PR is merged).

    Then create your model with the following command.

    ```bash
    docker build --no-cache --build-arg MHUB_MODELS_REPO=https://github.com/your_username/models-fork::branch -t dev/$model_name:latest .
    ```

    Again, you must change `https://github.com/your_username/models-fork` to the URL of your fork of our models repository and optionally specify the name of the branch separated by `::` in the URL.

    The command will create an image with the name `dev/$model_name:latest`. You can use any other name, but it is a good idea to organize your images.

4. Download the Sample Data

    Now, you need to download the sample data from IDC.

5. Run the Model

    Now that you have some sample data downloaded, you can run your model.
    Make sure to update the `/path/to/your/sample/data` and `/path/to/your/output/folder` with the paths to your sample data and an *empty* output folder.

    ```bash
    docker run dev/$model_name:latest -v /path/to/your/sample/data:/app/data/input_data:ro -v /path/to/your/output/folder:/app/data/output_data 
    ```

6. Inspect the Output

    Now you can inspect the output of your model. If you are satisfied with the output (e.g., the output looks as expected from the model or algorithm you are deploying to MHub), you can proceed to the next step.

    - Ensure that all files you want to be exported are present in the output folder. You can change how files generated internally from your MHub workflow are exported in the [DataOrganizer](../mhubio/mhubio_modules.md#dataorganizer) module in your [workflow's config.yml file](../mhubio/the_mhubio_config_file.md).

    - Inspect the content of **every** file generated and ensure any prediction generated reflects the expected accuracy as reported in the publication in the context of the selected test case.

    - Ask yourself, if you were to run the algorithm the very first time without any knowlege beyond what is provided in the [model card](../mhub_models/model_json.md), is the output you are seeing what you would expect, useful, transparent and simple to understand?

7. Prepare the Test Results

    In order for us to verify your test results, we need to know the sample data you choose to run your model on as well as the output your model produced.

    6.1. Zip your models output folder.

    ```bash
    zip -r /path/to/your/output/folder.zip /path/to/your/output/folder
    ```

    6.2. Upload the zip file to a publicly accessible location (e.g. GitHub, Google Drive, Dropbox, etc.).

8. Submit your Test Results

    After you have successfully tested your model and ensured that it delivers the expected results for all sample data, you can send us your test results.

    All that's left to do is to report your test results back to us. Therefore, please update your submission pull request with a comment in which you select one of the two suggested templates, depending on whether your workflow starts from a single Dicom input (e.g., starting with a [DicomImporter](../mhubio/mhubio_modules.md#dicomimporter)) Module) or non-dicom files or multiple inputs (e.g. starting with a [FileStructureImporter](../mhubio/mhubio_modules.md#filestructureimporter) module).

    You can find further instructions on how to collect the `IDC Version`, `SeriesInstanceUID` and `AWS URL` for your test data cases [here](https://github.com/MHubAI/models/pull/47#issuecomment-1870640491).

    For workflows starting with a DicomImporter:

    ```markdown
    ## Test Results

    general:
    - IDC Verson: <enter IDC version>
    - Output: <provide zipped output folder if model run on all cases>

    case 1: 
    - SeriesInstanceUID: <enter SeriesInstanceUID of your test case 1>
    - AWS URL: <enter URL to your test case 1>
    - Output: <provide zipped output folder per case if model run per case>

    case 2:
    ...

    comments:
    <write any additional comments here>
    ```

    For workflows starting with a FileStructureImporter:

    ```markdown

    general:
    - IDC Verson: <enter IDC version>
    - Output: <provide zipped output folder if model run on all cases>

    case 1 - input 1: 
    - SeriesInstanceUID: <enter SeriesInstanceUID of your test case 1>
    - AWS URL: <enter URL to your test case 1>
    - Input: <the relative path where the instance is loaded into, e.g. `case1/ct`>

    case 1 - input 2:
    - SeriesInstanceUID: <enter SeriesInstanceUID of your test case 1>
    - AWS URL: <enter URL to your test case 1>
    - Input: <the relative path where the instance is loaded into, e.g. `case1/mr`>

    case 2 - input 1:
    ...

    comments:
    <write any additional comments here>
    ```
