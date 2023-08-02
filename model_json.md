# Model Metadata JSON

This document describes the JSON format used to store model metadata for all MHub models.
A `meta.json` is required for each MHub model.

The document structure is mainly aligned to the model card approach proposed by Mitchell, M. et al. (2019).

All fields not marked as optional are required.

```json
{
    "id": "my_model",       // str: a unique mhub model id ([\w\_]+)
    "title": "My Model",    // str: a human readable title, used to display the model on the website
    "summary": {},          // obj: a summary of the model
    "details": {},          // obj: according to Model Card's first "Model Details" section
    "info": {}              // obj: collection of additional Model Card sections 
}
```

## ID

The ID of a model is an alphanumeric string that uniquely identifies an MHub model. The ID must not contain spaces or other special characters except underscores. IDs are case insensitive and should be all lowercase. The ID must be meaningful. We recommend that model names be preceded by the manufacturer's abbreviation to avoid name collisions. If models have a common component, such as a single model trained for multiple use cases, they can be added as separate components. Components should be separated by an underscore. Avoid using underscores as part of the vendor name, model, or use case. A version should not be added for the first release on MHub, but only for subsequent releases that drastically change the functionality of the model. Components must be ordered from most to least generalizable.

Example: `vendor_model_usecase_version`.

## Title

The title th emodel will be displayed under on MHub's website and other integrations (e.g., out MHub 3D Slicer plugin). Choose a meaningful and concise name.

## Summary

```json
"/summary": {
    "inputs": [],
    "outputs": [],
    "model": {},
    "data": {}
}
```

### Inputs

Describe all inputs of the model.

***NOTE**: This section is work-in-progress and subject to changes. We need to define attributes per input type (image, data, ..). Below is an example of the current, static structure.*

```json
"inputs": {
    "modality": {
        "label": "Imaging Modality",
        "text": "CT"
    },
    "bodypartexamined": {
        "label": "Body Part Examined",
        "text": "Chest"
    },
    "slicethickness": {
        "label": "Slice Thickness",
        "text": "2.5mm"
    },
    "contrast": {
        "label": "Contrast Agent",
        "text": "No"
    }
}
```

### Outputs

Describe all outputs of the model.

***NOTE**: This section is work-in-progress and subject to changes. We need to define attributes per data type (segmentation file, scores in a json or csv, ...). Below is an example of the current, static structure.**

```json
"outputs": {
    "type": {
        "label": "Type",
        "text": "Segmentation"
    },
    "classes": {
        "label": "Segmented Classes",
        "text": "LEFT_UPPER_LUNG_LOBE, LEFT_LOWER_LUNG_LOBE, RIGHT_UPPER_LUNG_LOBE, RIGHT_MIDDLE_LUNG_LOBE, RIGHT_LOWER_LUNG_LOBE"
    }
}
```

### Model

```json
"/summary/model": {
    "architecture": "", // str: the model architecture (e.g. ResNet50)
    "training": "",     // str: the training method (e.g. supervised)
    "approach": "",     // str: the approach (e.g. 3D)
}
```

***TODO: we need to define some guideline / default values. E.g. for training: supervised, semi-supervised, unsupervised, transfer learning, ... And revisit the keys, aproach is very broad and might invite for explainations, however we need tag-style keywords here.***

### Data

Describe the data used to train and evaluate the model.

```json
"/summary/data": {
    "training": 0,      // int:  number of samples (e.g. patients) used during training
    "evaluation": 0,    // int:  number of samples used for evaluation
    "public": true,     // bool: wheather the model was evaluated on public data
    "external": false   // bool: wheather the model was cross-validated 
}
```

## Details

This section describes various details of the AI pipeline, the author, the maintaining team, references to the publication and the code repository.

```json
"/details": {
    "name": "",             // str:       the original name of the model (e.g., as used in publications)
    "version": "1.0.0",     // str:       the version of the model (x.y.z)
    "devteam": "",          // str:       the lab or development team of the model
    "authors": [],          // list[str]: the authors of the model
    "type": "",             // str:       short description of the model type (e.g., Relational two-stage U-Net (Cascade of two relational U-Net, trained end-to-end)
    "date": {
        "code": "dd.mm.yyyy",      // str: the date the code was published
        "weights": "dd.mm.yyyy",   // str: the date the weights were published
        "pub": "dd.mm.yyyy"        // str: the date the publication was published
    },
    "cite": "",             // str:       the citation of the publication (APA)
    "licence": {
        "code": "",         // str: the licence of the code
        "weights": "",      // str: the licence of the weights
    },
    "publications": [{      // list[obj]: list of publications (one required)
        "title": "",        // str: the full title of the publication
        "uri": "",          // str: the url of the publication
    }],
    "github": "",           // str: the url of the github repository
}
```

## Info

```json
"info": {
    "use": {},
    "factors": {},
    "metrics": {},
    "evaluation": {},
    "training": {},
    "analyses": {},
    "ethics": {},
    "limitations": {}
}
```

Additional sections of the Model Card are bundled under info in the model json.
These sections contain detailed information and descriptions about the model's use case, how to use the model, the model's data compatibility, and additional information about the training process and evaluation results.

```json
"/info/*": {
    "text": "",         // str: the text of the section
    "references": [{    // list[str]: optional list of references (e.g., links)
        "label": "",    // str: the label of the reference
        "uri": ""       // str: the url of the reference
    }],
    "tables": [{        // list[obj]: optional list of tables
        "label": "",    // str: the label of the table
        "entries": {}   // obj[str->str]: table rows, the key will appear in the first, the value in the second column
    }]
}
```

Each section can be described with free text that is a minimum of 50 words and a maximum of 500 words. Text formatting (e.g., bold, italics, underlines, lists) is not allowed.

Links must not be included in the text, but may be added as references instead. Please reference the links in your text using the `[1]` notation, where the number in the brackets corresponds to the position indicated in the `references` array.

To display evaluation results or data statistics, you can add up to two tables per section.
For each table you can define a `label` that will be displayed above the table. You can add up to 10 entries per table. Each entry must have a `key` and a `value`. The `key` is displayed in the first column, the `value` in the second column. If you have less than 2 entries, you should do without a table and describe your results in the text of the respective section.

The following sections are mandatory:

- [x] Indended Use
- [x] Factors
- [x] Metrics
- [x] Evaluation Data
- [x] Training Data
- [x] Quantitative analysis
- [x] Ethical considerations
- [x] Caveats and Recommendations

The following sections are optional:  
***To be discussed; in case we want to have optional sections***
