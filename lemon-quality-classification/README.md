# lemon-quality-classification

[Kaggle - Lemon Quality Dataset](https://www.kaggle.com/datasets/yusufemir/lemon-quality-dataset) を OCI Vision + Data Labeling を用いて実施するサンプル

## how to use

<!-- @import "[TOC]" {cmd="toc" depthFrom=3 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [データの準備](#データの準備)
- [Object Storage へのアップロード](#object-storage-へのアップロード)
- [Data Labeling を用いたラベル付け](#data-labeling-を用いたラベル付け)
- [Vision - カスタム・モデルの作成](#vision---カスタムモデルの作成)
- [カスタム・モデルを用いた画像分類](#カスタムモデルを用いた画像分類)

<!-- /code_chunk_output -->

### データの準備

データをダウンロードし、`lemon-quality-classification/` に展開します。

```bash
$ tree lemon-quality-classification/ -L 2
lemon-quality-classification/
├── README.md
├── lemon_dataset # レモンのデータセット
│   ├── bad_quality # 品質の悪いレモンの画像セット
│   ├── empty_background # 空背景の画像セット
│   └── good_quality # 品質の良いレモンの画像セット
├── lemon_quality.ipynb # カスタムモデルを使うための検証用ノートブック
└── parameters # OCI CLI 用のパラメータ
```

### Object Storage へのアップロード

学習データを格納するための Object Storage - Bucket を作ります。

```bash
NAMESPACE_NAME=<your-object-storage-namespace>
BUCKET_NAME=<object-storage-bucket-name>
COMPARTMENT_ID=<your-compartment-id>

oci os bucket create \
    --namespace-name $NAMESPACE_NAME \
    --name $BUCKET_NAME \
    --compartment-id $COMPARTMENT_ID
```

学習用データを Bucket に格納するための設定を行います。`utils/config.py` を自身の環境に合わせて修正します。(`bad_quality`, `empty_background`, `good_quality`それぞれに対して実施します)

修正例:

```py
# config file path
CONFIG_FILE_PATH = "~/.oci/config"
# config file profile
CONFIG_PROFILE = "DEFAULT"
# identifier
REGION_IDENTIFIER = "ap-tokyo-1"
# ... omit ...
OBJECT_STORAGE_PREFIX = "bad_quality/"
# Files present inside this directory will be uploaded to the object storage bucket
DATASET_DIRECTORY_PATH = f"/home/shukawam/work/oci-vision-dls-examples/lemon-quality-classification/lemon_dataset/bad_quality"
# Object storage bucket name where the dataset will be uploaded
OBJECT_STORAGE_BUCKET_NAME = "lemon-quality-classification-training-data"
# Namespace of the object storage bucket
OBJECT_STORAGE_NAMESPACE = "orasejapan"
```

ファイルアップロード用のスクリプトを実行します。

```bash
python3 utils/upload_files_script.py
```

以下のように出力されれば OK です。

```bash
# ... omit ...
Successfully uploaded 452 file(s)
Failed to upload 0 file(s)
Finished in 8.46 second(s)
```

### Data Labeling を用いたラベル付け

次に、Data Labeling でデータセットを作成するために `lemon-quality-classification/parameters/create-dls-dataset.json` を修正します。\<your-compartment-id\>, \<your-bucket-name\>, \<your-object-storage-namespace\> をご自身の環境に合わせて修正してください。

```json
{
  "annotationFormat": "SINGLE_LABEL",
  "compartmentId": "<your-compartment-id>",
  "datasetFormatDetails": {
    "formatType": "IMAGE"
  },
  "datasetSourceDetails": {
    "bucket": "<your-bucket-name>",
    "namespace": "<your-object-storage-namespace>",
    "sourceType": "OBJECT_STORAGE"
  },
  "description": "Dataset for lemon quality classification.",
  "displayName": "lemon-quality-classification-dataset",
  "labelSet": {
    "items": [
      {
        "name": "bad_quality"
      },
      {
        "name": "empty_background"
      },
      {
        "name": "good_quality"
      }
    ]
  }
}
```

以下のように実行し、データセットを作ります。

```bash
oci data-labeling-service dataset create \
    --from-json file://lemon-quality-classification/parameters/create-dls-dataset.json
```

次に、Data Labeling でデータレコードを生成するために `lemon-quality-classification/parameters/generate-dataset-record.json` を修正します。\<your-dataset-id\> をご自身の環境に合わせて修正してください。

```json
{
  "datasetId": "<your-dataset-id>"
}
```

以下のように実行し、データレコードを生成します。

```bash
oci data-labeling-service dataset generate-dataset-records \
  --from-json file://lemon-quality-classification/parameters/generate-dataset-record.json
```

レコードの生成が完了後に、一括でラベルを付けるために `utils/config.py`, `utils/bulk_labeling_script.py` を修正します。

`utils/config.py` で、\<your-dataset-id\> をご自身の環境に合わせて修正してください。

```py
# ocid of the DLS Dataset
DATASET_ID = "<your-dataset-id>"
# ... omit ...
# Possible values for ANNOTATION_TYPE "BOUNDING_BOX", "CLASSIFICATION"
ANNOTATION_TYPE = "CLASSIFICATION"
```

`utils/classification_config.py` で以下のようになっていることを確認する

```py
# ... omit ...
# Possible values for labeling algorithm "FIRST_LETTER_MATCH", "FIRST_REGEX_MATCH", "CUSTOM_LABELS_MATCH"
LABELING_ALGORITHM = "CUSTOM_LABELS_MATCH"
# For CUSTOM_LABEL_MATCH specify the label map
LABEL_MAP = {"bad_quality/": ["bad_quality"], "empty_background/": ["empty_background"], "good_quality/": ["good_quality"]}
```

bulk 処理でラベルを付けていきます。

```bash
python3 utils/bulk_labeling_script.py
```

### Vision - カスタム・モデルの作成

OCI Vision のプロジェクトを作成するために、`lemon-quality-classification/parameters/create-vision-project.json` の \<your-compartment-id\> をご自身の環境に合わせて修正します。

```json
{
  "compartmentId": "<your-compartment-id>",
  "description": "custom model demo for image classification.",
  "displayName": "lemon-classification-project"
}
```

以下のように実行し、Vision のプロジェクトを作成します。

```bash
oci ai-vision project create \
  --from-json file://lemon-quality-classification/parameters/create-vision-project.json
```

カスタム・モデルを作成するために、`lemon-quality-classification/parameters/create-model.json` の \<your-compartment-id\>, \<your-project-ocid\>, \<your-dataset-ocid\> をご自身の環境に合わせて修正します。

```json
{
  "compartmentId": "<your-compartment-id>",
  "description": "custom model for lemon quality classification.",
  "displayName": "lemon-quality-classification-custom-model",
  "isQuickMode": true,
  "modelType": "IMAGE_CLASSIFICATION",
  "projectId": "<your-project-ocid>",
  "trainingDataset": {
    "datasetId": "<your-dataset-ocid>",
    "datasetType": "DATA_SCIENCE_LABELING"
  }
}
```

以下のように実行し、Vision のカスタム・モデルを作成します。（作成には、数時間を要します）

```bash
oci ai-vision model create \
  --from-json file://lemon-quality-classification/parameters/create-model.json
```

### カスタム・モデルを用いた画像分類

`lemmon_quality.ipynb` を参照ください。
