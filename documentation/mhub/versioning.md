# MHub Versioning

This document describes how validation, versioning, and marking of MHub models is performed to achieve a high level of reproducibility.

## Versions in the MHub Ecoverse

In the MHub ecoverse, we need to distinguish between different types of versions for MHub (v) and its dependencies, and for MHub models (m) themselves. Versions are incremented independently of each other. We increment the MHub version whenever we release a new version of MHub (see section [Stable Releases](versioning.md#stable-releases)). For models, we increment the model version whenever the model releases a new version or when the model is significantly updated. Since specifying the model version in the Docker file is mandatory for MHub models, a change in model version due to a model update always is always connected with an update of the model's Docker file in our [Models Repository](https://github.com/MHubAI/models/). Independent maintenance of Mhub and model versions means that there is a 1:1 relationship between an Mhub version and a model version, and exactly one version of the model is available for each stable Mhub version.

## Reproducibility Checks

All MHub models undergo a weekly reproducibility check. That is, we build the Dockerfile and check if the output of the model matches the expected output. We perform various checks, such as validating file names and file sizes, and comparing file contents such as Dice scores for segmentation images. Only models that pass a check are uploaded to Dockerhub. If a model cannot be built or does not pass all checks, we indicate this with a warning on our website. At the same time, we give the developers of the model the opportunity to fix the problem or explain that the model has changed significantly, resulting in a version increase. This means that you can always expect accurate results for all MHub versions, even if you are using the latest version of the model with the latest features (see section [Latest Release](versioning.md#latest-release)), even if no warning is displayed on our website.

Thanks to our regular model testing, you can trust all models on MHub. In the past, we found that an updated library that the model depended on caused the model to produce slightly different segmentations on each run. This is very hard to notice when you only run the model once, especially since the code and commit history of the model didn't change at all. However, through our regular reviews, we can detect such issues and prevent the model from being built until the bugs are fixed. If you have contributed a model to MHub and we detect an anomaly in your model, we will warn you accordingly. We encourage all users to report to us if they discover an anomaly in an MHub model so that we can review our controls and update or expand them as necessary.

## Releases

We provide two different kinds of releases for Mhub models: latest and stable releases.

### Latest Release

`:latest`

As explained in the Reproducibility Checks section above, we build all MHub models every week to ensure the model code stays up-to-date and the most recent modules and features (and fixes) of MHub are available to you. As long as the built image passes all of our output checks, we push it to DockerHub under the `:latest` tag.

### Stable Releases

`:stable` and `:v1-m1`

While `:latest` always contains the latest check-passing build for any MHub model based on weekly builds, field-tested feature sets and new features are added under a new MHub version. When a new version of MHub is released, we freeze the latest containers in time and tag them with a new version tag.

The following example shows an increase in MHub version from `v1` to `v2`. Note that the version of the model (m1, m2, ...) is maintained during a stable MHub version.

```text
mhubai/totalsegmentator:v1-m1  ->  mhubai/totalsegmentator:v2-m1
mhubai/platipy:v1-m2           ->  mhubai/platipy:v2-m2
mhubai/casust:v1-m1            ->  mhubai/casust:v2-m1
```

The containers for the two MHub versions (v1 and v2 in this example) are now frozen and do not change, ensuring high reproducibility. At the same time, all models of the same version are based on the same Mhub base image, which means that they all have the same modules and functions available. For example, we can change the configuration between two stable Mhub versions. However, for all models of the same version, all configurations remain consistent.

Stable releases are especially important if you want to integrate MHub with a third-party system. All models in a stable release use the same Mhub features and configuration syntax. To ensure compatibility with your implementation, it is advisable to implement all models with the latest version (e.g. `:v1-m1`). Once we release the next stable MHub version (e.g. `:v2-m1`), you should update and test your implementation with the new version in a test environment and then port it to your productive environment with the new version.

To always access the latest, stable release, you can use the `:stable` tag. To stick with the above example, the `:stable` tag refers to the following containers after the version update.

```text
mhubai/totalsegmentator:stable  =  mhubai/totalsegmentator:v2-m1
mhubai/platipy:stable           =  mhubai/platipy:v2-m2
mhubai/casust:stable            =  mhubai/casust:v2-m1
```
