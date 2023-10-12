# car-detection

[Kaggle - Car Object Detection](https://www.kaggle.com/datasets/sshikamaru/car-object-detection) を OCI Vision + Data Labeling を用いて実施するサンプル

## how to use

<!-- @import "[TOC]" {cmd="toc" depthFrom=3 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [データの準備](#データの準備)
- [Object Storage へのアップロード](#object-storage-へのアップロード)
- [Data Labeling を用いたラベル付け](#data-labeling-を用いたラベル付け)
- [Vision - カスタム・モデルの作成](#vision---カスタムモデルの作成)
- [カスタム・モデルを用いたオブジェクト検出](#カスタムモデルを用いたオブジェクト検出)

<!-- /code_chunk_output -->

### データの準備

データセットをダウンロードし、 `car-detection/` に展開します。

```bash
$ tree car-detection/ -L 2
car-detection/
├── README.md
├── car_detection.ipynb # カスタムモデルを使うための検証用ノートブック
└── data
    ├── sample_submission.csv # Kaggle のコンペ提出用の CSV のため、今回は使用しない
    ├── testing_images # テスト用データ
    ├── train_solution_bounding_boxes.csv # 学習用データのバウンディングボックス情報
    └── training_images # 学習用データ
```

### Object Storage へのアップロード

学習用データを格納するための Object Storage - Bucket を作ります。

```bash
NAMESPACE_NAME=<your-object-storage-namespace>
BUCKET_NAME=<object-storage-bucket-name>
COMPARTMENT_ID=<your-compartment-id>

oci os bucket create \
    --namespace-name $NAMESPACE_NAME \
    --name $BUCKET_NAME \
    --compartment-id $COMPARTMENT_ID
```

学習用データを Bucket に格納するための設定を行います。`utils/config.py` を自身の環境に合わせて修正します。

修正例:

```py
# config file path
CONFIG_FILE_PATH = "~/.oci/config"
# config file profile
CONFIG_PROFILE = "DEFAULT"
# identifier
REGION_IDENTIFIER = "ap-tokyo-1"
# ... omit ...
# Files present inside this directory will be uploaded to the object storage bucket
DATASET_DIRECTORY_PATH = f"/home/shukawam/work/oci-vision-dls-examples/car-detection/data/training_images"
# Object storage bucket name where the dataset will be uploaded
OBJECT_STORAGE_BUCKET_NAME = "car-detection-training-data"
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
Total files present in the given directory : 1001
Successfully uploaded 1001 file(s)
Failed to upload 0 file(s)
Finished in 21.21 second(s)
```

### Data Labeling を用いたラベル付け

次に、Data Labeling でデータセットを作成するために `car-detection/parameters/create-dls-dataset.json` を修正します。\<your-compartment-id\>, \<your-bucket-name\>, \<your-object-storage-namespace\> をご自身の環境に合わせて修正してください。

```json
{
  "annotationFormat": "BOUNDING_BOX",
  "compartmentId": "<your-compartment-id>",
  "datasetFormatDetails": {
    "formatType": "IMAGE"
  },
  "datasetSourceDetails": {
    "bucket": "<your-bucket-name>",
    "namespace": "<your-object-storage-namespace>",
    "sourceType": "OBJECT_STORAGE"
  },
  "description": "Dataset for car detection.",
  "displayName": "car-detection-dataset",
  "labelSet": {
    "items": [
      {
        "name": "car"
      }
    ]
  }
}
```

以下のように実行し、データセットを作ります。

```bash
oci data-labeling-service dataset create \
    --from-json file://car-detection/parameters/create-dls-dataset.json
```

次に、Data Labeling でデータレコードを生成するために `car-detection/parameters/generate-dataset-record.json` を修正します。\<your-dataset-id\> をご自身の環境に合わせて修正してください。

```json
{
  "datasetId": "<your-dataset-id>"
}
```

以下のように実行し、データレコードを生成します。

```bash
oci data-labeling-service dataset generate-dataset-records \
  --from-json file://car-detection/parameters/generate-dataset-record.json
```

レコードの生成が完了後に、Kaggle のデータセットに含まれている `train_solution_bounding_boxes.csv` を用いて一括でラベル付けを行うために必要なファイルを生成します。

`convert_config.py` に含まれる `BOUNDING_BOXES_PATH`, `OUTPUT_FILE` をご自身の環境に合わせて修正します。

```py
# maximum number of DLS Dataset records that can be retrieved from the list_records API operation for labeling
# limit=1000 is the hard limit for list_records
LIST_RECORDS_LIMIT = 1000
# an array where the elements are all of the labels that you will use to annotate records in your DLS Dataset with.
# Each element is a separate label.
LABEL = ['car']
# Path of bounding_boxes_path
BOUNDING_BOXES_PATH = "/home/shukawam/work/oci-vision-dls-examples/car-detection/data/train_solution_bounding_boxes.csv"
# Oath of output file
OUTPUT_FILE = '/home/shukawam/work/oci-vision-dls-examples/car-detection/input_data.csv'
COLUMNS = ['record_id', 'x1', 'x2', 'x3',
           'x4', 'y1', 'y2', 'y3', 'y4', 'label']
DROPED_COLUMNS = ['name', 'image', 'compartment_id', 'dataset_id', 'is_labeled', 'lifecycle_state', 'time_created', 'time_updated',
                  'record_metadata.depth', 'record_metadata.height', 'record_metadata', 'record_metadata.record_type', 'record_metadata.width']
IMG_H, IMG_W = (380, 676)
```

`config.py` に含まれる `DATASET_ID` をご自身の環境に合わせて修正します。

```py
# ocid of the DLS Dataset
DATASET_ID = "ocid1.datalabelingdataset.oc1.ap-tokyo-1.amaaaaaassl65iqafkfvmbrzeli5xxmradunv7zbtqzp5z4jgfg3v7xf7yfq"
```

以下のように実行し、Kaggle - Car Object Detection から Data Labeling のフォーマットに変換します。

```bash
python3 utils/convert_dls_format.py
```

変換に成功すると、`OUTPUT_FILE` で指定したファイルに以下のような内容が出力されます。

```csv
record_id,x1,x2,x3,x4,y1,y2,y3,y4,label
ocid1.datalabelingrecord.oc1.ap-tokyo-1.amaaaaaassl65iqalika6kf3tgxkrdvostjg6atsxtu4rxs3zvhssasa4xiq,0.010924981791420119,0.010924981791420119,0.1973780044378698,0.1973780044378698,0.5063552460526316,0.5063552460526316,0.6242308936842105,0.6242308936842105,['car']
ocid1.datalabelingrecord.oc1.ap-tokyo-1.amaaaaaassl65iqac7cqecsyyxxi2sdmadrxm4dems5bg7psuq2cgqwn7i7q,0.0,0.0,0.20975965044378697,0.20975965044378697,0.4428837434210527,0.4428837434210527,0.6294122410526316,0.6294122410526316,['car']
ocid1.datalabelingrecord.oc1.ap-tokyo-1.amaaaaaassl65iqaohl3wfresb4njtl67xa7o72viax3piluwasludgyaxwq,0.3277494536982249,0.3277494536982249,0.5156591405325444,0.5156591405325444,0.48044851026315794,0.48044851026315794,0.626821567368421,0.626821567368421,['car']
```

これを元に bulk 処理でバウンディングボックスを付けていきます。`config.py` で以下のように設定します。

```py
# Type of Annotation
# Possible values for ANNOTATION_TYPE "BOUNDING_BOX", "CLASSIFICATION"
ANNOTATION_TYPE = "BOUNDING_BOX"
```

以下のように実行します。

```bash
python3 utils/bulk_labeling_script.py
```

### Vision - カスタム・モデルの作成

OCI Vision のプロジェクトを作成するために、`car-detection/parameters/create-vision-project.json` の \<your-compartment-id\> をご自身の環境に合わせて修正します。

```json
{
  "compartmentId": "<your-compartment-id>",
  "description": "custom model demo for object detection.",
  "displayName": "car-detection-project"
}
```

以下のように実行し、Vision のプロジェクトを作成します。

```bash
oci ai-vision project create \
  --from-json file://car-detection/parameters/create-vision-project.json
```

カスタム・モデルを作成するために、`car-detection/parameters/create-model.json` の \<your-compartment-id\>, \<your-project-ocid\>, \<your-dataset-ocid\> をご自身の環境に合わせて修正します。

```json
{
  "compartmentId": "<your-compartment-id>",
  "description": "custom model for car object detection.",
  "displayName": "car-detection-custom-model",
  "isQuickMode": false,
  "modelType": "OBJECT_DETECTION",
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
  --from-json file://car-detection/parameters/create-model.json
```

### カスタム・モデルを用いたオブジェクト検出

`car_detection.ipynb` を参照ください。
