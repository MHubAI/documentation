# Handson Session

1. Install dependencies

```bash
# install pipx 
sudo apt install pipx
pipx ensurepath

# install idc index
pipx install idc-index
```

2. Define environment variables

```bash
export input_dir=$(realpath -m ~/Desktop/mhub_tutorial/data)
export output_dir=$(realpath -m ~/Desktop/mhub_tutorial/output)
```

3. Setup Directories

```bash
mkdir -p $input_dir $output_dir
```

4. Download IDC data

In this tutorial we use two chest CT images from case 100005 with the following SeriesInstanceUIDs:

- `1.2.840.113654.2.55.17324290215190661437113320769488297837`
- `1.2.840.113654.2.55.212474527960092593758511400493563127113`

You can download a single series with the following command:

```bash
idc download-from-selection \
  --series-instance-uid 1.2.840.113654.2.55.17324290215190661437113320769488297837,1.2.840.113654.2.55.212474527960092593758511400493563127113 \
  --show-progress-bar true \
  --download-dir $input_dir
```

Or use the IDC portal to browse for data, generate a [manifest](manifest_20250413_185954_aws.s5cmd) and download it with idc-index.

```bash
idc download-from-manifest \
  --manifest manifest_20250413_185954_aws.s5cmd \
  --show-progress-bar true \
  --download-dir $input_dir
```

5. Run MHub.ai models

```bash
# run totalsegmentator
docker run --rm -t --gpus all --network=none -v $input_dir:/app/data/input_data:ro -v $output_dir:/app/data/output_data mhubai/totalsegmentator:latest --workflow default

# run lungmask
docker run --rm -t --gpus all --network=none -v $input_dir:/app/data/input_data:ro -v $output_dir:/app/data/output_data mhubai/lungmask:latest --workflow default

# run lunglobes
docker run --rm -t --gpus all --network=none -v $input_dir:/app/data/input_data:ro -v $output_dir:/app/data/output_data mhubai/gc_lunglobes:latest --workflow default
````