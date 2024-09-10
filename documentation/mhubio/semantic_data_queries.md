# Semantic Data Type Query (DTQ)

A special feature of our MHub- IO framework is that it abstracts the file management of all imported and generated files and data during the execution of an MHub workflow. This means you never have to worry about where a file is stored (or how it should be named). Instead, for each file you import or create, you specify the file type along with a metadatum based on key values. To access files, for example for a conversion step or to export them with our DataOrganizer module, you specify a semantic data query (DTQ). With DTQs, you can access files and data by their file type or file name and associated metadata.

## Semantic Data Annotation

Semantic data is expressed in terms of metadata-key-value pairs, which are stored internally as `Meta` instances. Metadata has a string representation which is a `:` separated list of `key=value` pairs, e.g. `key1=value1:key2=value2`.

Semantic data annotations are available for all files and data passed between MHub- IO modules. For files, the semantic data annotation starts with the string representation of the file type (e.g. dicom or nifti), followed by the string representation of the metadata. For data, the semantic data annotation starts with the file name, followed by the metadata representation. The file type or file name is separated from the metadata string representation by a `:`.

## DTQ Syntax

The simplest form of a DTQ is a direct match identifier. For example, consider the following store:

```text
dicom:mod=ct
nrrd:mod=ct
nrrd:mod=seg
```

To retrieve the file CT in DICOM format, a DTQ corresponding to the data type and the entire metadata is sufficient: `dicom:mod=ct` retrieves exactly the first file in the above list. This means that all generated data can be queried uniquely only if their metadata representation is unique for the file type or file name.

However, you can also use DTQ to retrieve a group of data. The simplest example of this use case is to specify incomplete metadata. For example, DTQ `nrrd` retrieves all nrrd files regardless of their metadata, DTQ `any:mod=ct` retrieves all files or data with metadata `mod=ct` regardless of their file type or file name. Note that the keyword `any` is a placeholder for any file type or file name.

### Value Placeholder

The value placeholder `*` can be used to select all files or data that have a meta attribute, regardless of its value. The keyword `any` can be used as a wildcard to find any file type or data name.

For example, the DTQ `nrrd:mod=*` retrieves all nrrd files that have any value specified under the `mod` key, the DTQ `any:model=test` retrieves all files or data, regardless of their file type or file name, where the metadata value for model is test.

### Query Chaining

More complex queries can be created by concatenating DTQs with the operators `AND`, `OR`, and ` NOT `. For example, the DTQ `nrrd:mod=ct OR nifti:mod=seg` finds all nrrd or nifti files that have the metadate `mod=ct`. The DTQ `nrrd:mod=* AND NOT nrrd:mod=seg` finds all nrrd files that have any value specified under the `mod` key, except for the `seg` value.

For more complex queries, parentheses can be used, e.g. `nrrd AND (any:mod=seg OR any:mod=ct)` finds all files of type nrrd whose metadata attribute is either `seg` or `ct`.

### Value Pools

To include multiple values into a query, e.g., all data of modality MR and CT you can use the in-place or operator `|`. For example, the DTQ `nrrd:mod=ct|mr` retrieves all nrrd files that have the metadata attribute `mod` set to either `ct` or `mr`. The DTQ `nrrd|nifti:mod=seg` retrieves all nrrd or nifti files that have the metadata attribute `mod` set to `seg`.

## Value Operators

So far we have seen examples where the `=` operator has been used to match metadata values exactly. DTQ supports a whole set of additional operators for matching and comparison. Some operators behave differently depending on the data type of the value stored under a key (e.g. sting, number, list).

The following table lists all supported operators:

- `=`: Exact match
- `!=`: Not equal
- `>`: Greater than (only for numbers)
- `>=`: Greater than or equal (for numbers and lists)
- `<`: Less than (only for numbers)
- `<=`: Less than or equal (for numbers and lists)
