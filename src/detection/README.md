# Object Detection

## Overview

This guide shows how to train a model on a tiny dataset containing ships, and then how to make predictions on a TIFF file. This makes use of a set of scripts utilizing the Tensorflow Object Detection API, and AWS Batch to run jobs. Unless otherwise stated, all commands should be run from inside the Docker CPU container relative to `/opt/src/detection`. In addition, the paths are with reference to the file system of the Docker CPU container. Note that there are bugs, so make sure to check the issues with the `object-detection` and `bug` labels.

## Data Prep

First, download the raw ships data from  `s3://raster-vision/datasets/detection/singapore_ships.zip` to `/opt/data/datasets/detection/singapore_ships.zip` and unzip it.

In order to train a model, you must convert a GeoTIFF and GeoJSON file with annotations into training chips and a CSV file representing bounding boxes in the chips' frame of reference. To do this for the first file in the dataset, run this:

```
python scripts/tiff_chipper.py \
    --tiff-path /opt/data/datasets/detection/singapore_ships/0.tif \
    --json-path /opt/data/datasets/detection/singapore_ships/0.geojson \
    --output-dir /opt/data/datasets/detection/singapore_ships_chips_tiny \
    --chip-size 300 --debug
```

Create a `label_map.pbtxt` file in the `singapore_ships_chips_tiny` directory. It should have a single entry with `id=1` and `name=Ships`. An example of this kind of file can be found in `pet_label_map.pbtxt`. Then, convert the output to TFRecord format, which is the required format for the TF Object Detection API. The files will be placed in the `singapore_ships_chips_tiny` directory

```
python scripts/create_tf_record.py \
    --data-dir /opt/data/datasets/detection/singapore_ships_chips_tiny \
    --label-map-path /opt/data/datasets/detection/singapore_ships_chips_tiny/label_map.pbtxt \
    --output-dir /opt/data/datasets/detection/singapore_ships_chips_tiny
```

When this is done, zip the `singapore_ships_chips_tiny` folder into `singapore_ships_chips_tiny.zip` and upload to S3 inside `s3://raster-vision/datasets/detection`.

## Training a model on EC2

The training algorithm is configured in a file, an example of which can be seen at `configs/ships/ssd_mobilenet_v1.config`. This file needs to be modified if the local paths for data files change, or to tweak the hyperparameters or model architecture.
Once this file is committed to a branch (with name denoted by `branch-name`) and pushed to the remote repo, start a training job on AWS Batch, by running the following *from the VM*. Note the quotes around the command to run on the container.
```
src/detection/scripts/batch_submit.py  <branch-name> \
    "/opt/src/detection/scripts/train_ec2.sh \
    --config-path /opt/src/detection/configs/ships/ssd_mobilenet_v1.config \
    --train-id ships1 \
    --dataset-id singapore_ships_chips_tiny \
    --model-id ssd_mobilenet_v1_coco_11_06_2017"
```

This requires that there is a model checkpoint file (for using a pre-trained model) at `s3://raster-vision/datasets/detection/ssd_mobilenet_v1_coco_11_06_2017.zip`. This file can be downloaded from the website for the TF Object Detection API.

Every 10 minutes, model checkpoints are synced to the S3 bucket under `results/detection/train/ships1`, since `ships1` is the `train_id`.
You can monitor training using Tensorboard by pointing your browser at `http://<ec2 instance ip>:6006`. It may take a few minutes before results show up. When you are done training the model (ie. after the total loss flattens out), you need to kill the Batch job since it's running in an infinite loop. If this script fails due to instance shutdown and is run again, it should pick up where it left off using saved checkpoints.

## Making predictions on EC2

After training finishes, you can make predictions for a GeoTIFF file. To do this, first upload the image to `s3://raster-vision/results/detection/predict/ships_2/image.tif`.
To start a prediction job, run the following command with the `checkpoint-id` set to the id in the filename of the latest training checkpoint file, which can be found in `s3://raster-vision/results/detection/ships1/train/train`.
```
src/detection/scripts/batch_submit.py <branch-name> \
    "/opt/src/detection/scripts/predict_ec2.sh \
    --config-path /opt/src/detection/configs/ships/ssd_mobilenet_v1.config \
    --train-id ships1 \
    --checkpoint-id 64336 \
    --tiff-id ships_2 \
    --dataset-id singapore_ships_chips_tiny"
```

You may want to run this with the `--cpu` flag to `batch_submit.py`, since the speedup from the GPU might be negligible. When this is finished running, there should be a GeoJSON file with predictions at `s3://raster-vision/results/detection/predict/ships_2/output/predictions.geojson`.

## Debugging

To debug, it will be helpful to run the above two scripts locally. This can be done by copying the necessary files from S3 and then running the scripts with the `--local` flag. These two scripts call other scripts in turn, and you will probably want to run them individually. When using the `--local` flag with the `predict_ec2.py` script, temporary files will be placed in `/opt/data/temp` which may be helpful for debugging.
