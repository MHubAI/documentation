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

1. Prepare your submission.

    The testing procedure takes a bit of time on your and our side, so please make sure to test and submit the right code.

    1.1. Ensure that your PR is up to date with our `main` branch.  
    1.2. Commit all changes to your local repository.  
    1.3. Push all changes to your remote fork.

2. Get the latest MHub base image.

    Ideally, you're base image is up to date during development. However, to ensure you have the latest base image, refetch it from Docker Hub.

    ```bash
    docker pull mhubai/base:latest
    ```

3. Build your model.

    To build your model, copy the repository url  (including the `.git`extension) of the fork repository where your mhub implementation currently is and your branch name.

    ```bash
    MHUB_MODEL_NAME="my_model"
    MHUB_MODELS_REPO="https://github.com/MHubAI/models.git"
    MHUB_MODELS_BRANCH="main"
    ```

    Then, run the following build command to build your model.

    ```bash
    docker build --no-cache -t mhubai-dev/$MHUB_MODEL_NAME:latest --build-arg MHUB_MODELS_REPO=$MHUB_MODELS_REPO::$MHUB_MODELS_BRANCH $MHUB_MODELS_REPO#$MHUB_MODELS_BRANCH:models/$MHUB_MODEL_NAME/dockerfiles
    ```

    When you build your model now, you need to reference the MHub implementation code. Since your model is not yet submitted, we cannot automatically pull your implementation from our models repository as we normally would. Therefore, you need to specify the `--build-arg MHUB_MODELS_REPO=` argument to provide the URL of the form you used to create the PR (this is the repository where your model resides until the PR is merged).

4. Prepare a test folder.
  
    Create a new folder on your local machine where you can store the sample data and the output of your model.

    ```bash
    MHUB_TEST_DIR=/path/to/your/test/folder
    MHUB_WORKFLOW_NAME="default"

    mkdir -p $MHUB_TEST_DIR $MHUB_TEST_DIR/$MHUB_WORKFLOW_NAME/sample $MHUB_TEST_DIR/$MHUB_WORKFLOW_NAME/reference
    ```

    Repeat this step for every workflow `MHUB_WORKFLOW_NAME` your model contains.
    *Note: every config file you ship in your model's configs folder is a separate workflow. The workflow name is the name of the config file without the `.yml` extension.*

5. Download the Sample Data

    Now, you need to download the sample data from IDC. To learn more about how to download sample data form IDX, please refer to the [IDC User Guide](https://learn.canceridc.dev/data/downloading-data).

    The following example uses the `idc-index` cli tool that can be downloaded with [pipx](https://pipx.pypa.io/stable/).

    ```bash
    # install idc-index cli
    pipx install idc-index

    # specify the SeriesInstanceUID and the download directory
    MHUB_TEST_SID=1.2.840.113654.2.55.257926562693607663865369179341285235858
    MHUB_TEST_SID_DIR="dicom"
  
    # download sample data
    idc download-from-selection \
      --series-instance-uid $MHUB_TEST_SID \
      --show-progress-bar true \
      --download-dir $MHUB_TEST_DIR/$MHUB_WORKFLOW_NAME/sample/$MHUB_TEST_SID_DIR \
      --dir-template ""
    ```

    Repeat this step for every sample you want to download. If required, you can also include files from other sources into the `$MHUB_TEST_DIR/$MHUB_WORKFLOW_NAME/sample` folder.
    Repeat this step for every workflow `MHUB_WORKFLOW_NAME` your model contains.

6. Run the Model

    Now that you have some sample data downloaded, you can run your model.

    ```bash
    MHUB_OUTPUT_DIR=/path/to/your/output/folder
    docker run mhubai-dev/$MHUB_MODEL_NAME:latest \
      --gpus all \
      -v $MHUB_TEST_DIR/$MHUB_WORKFLOW_NAME/sample/:/app/data/input_data:ro \
      -v $MHUB_TEST_DIR/$MHUB_WORKFLOW_NAME/reference:/app/data/output_data  \
      -w $MHUB_WORKFLOW_NAME
    ```

    Repeat this step for every workflow `MHUB_WORKFLOW_NAME` your model contains.

7. Inspect the Console Output

    MHub captures all `print()` statements in log files and displays a clean process overview on the console. Make sure that no uncaptured output is generated in your implementation (uncaptured output can generate repeated lines, omitted lines, or additional text that should not occur). If your implementation does not generate clean output, your model cannot be accepted.

    **Note**: Some Python packages contain print statements in `__init__.py` files or at file level in otherwise imported files that are executed at import time. However, in the MHUb workflow, we can only capture the console output during the actual execution (i.e. within the `task()` method of a [module](../mhubio/how_to_write_an_mhubio_module.md#the-task-method)). You can solve this problem by moving the import statements into the `task()` method of your module or by wrapping your implementation in a cli-script and then using [self.subprocess](../mhubio/how_to_write_an_mhubio_module.md#running-a-subprocess-from-a-module) to execute that cli-script.

8. Inspect the File Output

    Now you can inspect the output of your model. If you are satisfied with the output (e.g., the output looks as expected from the model or algorithm you are deploying to MHub), you can proceed to the next step.

    - Ensure that all files you want to be exported are present in the output folder. You can change how files generated internally from your MHub workflow are exported in the [DataOrganizer](../mhubio/mhubio_modules.md#dataorganizer) module in your [workflow's config.yml file](../mhubio/the_mhubio_config_file.md).

    - Inspect the content of **every** file generated and ensure any prediction generated reflects the expected accuracy as reported in the publication in the context of the selected test case.

    - Ask yourself, if you were to run the algorithm the very first time without any knowledge beyond what is provided in the [model card](../mhub_models/model_json.md), is the output you are seeing what you would expect, useful, transparent, and simple to understand?

9. Prepare the Test Results

    In order for us to verify your test results, we need to know the sample data you choose to run your model on as well as the output your model produced.

    9.1. Zip your models output folder.

    ```bash
    (cd $MHUB_TEST_DIR && zip -r - ./*) > test.zip
    ```

    9.2. Upload the zip file `test.zip` to [Zenodo](https://zenodo.org/) and obtain a public link to the file.

    9.3. Update the link in your `mhub.toml` file at the root of your model folder.

    ```toml
    [model.distibution]
    test = "https://zenodo.org/xxxxxxxxxxxxx"
    ```

10. Submit your Test Results

    After you have successfully tested your model and ensured that it delivers the expected results for all sample data, you can request a test run by creating a comment on your commit starting with `/test`.
