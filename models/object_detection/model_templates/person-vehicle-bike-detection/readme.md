# Person Vehicle Bike Detector

The crossroad-detection network model provides detection of three class objects: vehicle, pedestrian, non-vehicle (like bikes). This detector was trained on the data from crossroad cameras.

| Model Name | Complexity (GFLOPs) | Size (Mp) | AP @ [IoU=0.50:0.95] (%) | Links | GPU_NUM |
| --- | --- | --- | --- | --- | --- |
| person-vehicle-bike-detection-2000 | 0.82 | 1.84 | 16.5 | [snapshot](https://download.01.org/opencv/openvino_training_extensions/models/object_detection/v2/vehicle-person-bike-detection-2000-1.pth), [model template](./person-vehicle-bike-detection-2000/template.yaml) | 4 |
| person-vehicle-bike-detection-2001 | 1.86 | 1.84 | 22.6 | [snapshot](https://download.01.org/opencv/openvino_training_extensions/models/object_detection/v2/vehicle-person-bike-detection-2001-1.pth), [model template](./person-vehicle-bike-detection-2001/template.yaml) | 4 |
| person-vehicle-bike-detection-2002 | 3.30 | 1.84 | 24.8 | [snapshot](https://download.01.org/opencv/openvino_training_extensions/models/object_detection/v2/vehicle-person-bike-detection-2002-1.pth), [model template](./person-vehicle-bike-detection-2002/template.yaml) | 4 |
| person-vehicle-bike-detection-2003 | 6.78 | 1.95 | 33.6 | [snapshot](https://storage.openvinotoolkit.org/repositories/openvino_training_extensions/models/object_detection/v2/vehicle-person-bike-detection-2003.pth), [model template](./person-vehicle-bike-detection-2003/template.yaml) | 2 |
| person-vehicle-bike-detection-2004 | 1.88 | 1.95 | 27.4 | [snapshot](https://storage.openvinotoolkit.org/repositories/openvino_training_extensions/models/object_detection/v2/vehicle-person-bike-detection-2004.pth), [model template](./person-vehicle-bike-detection-2004/template.yaml) | 2 |

Average Precision (AP) is defined as an area under the precision/recall curve.

Please note that despite the comparable complexity of 2001 and 2004 models the first one is well optimized in the IR format. It shows great inference speed that compensate the accuracy difference between these two models. So, if you are focused on the inference speed, do not neglect the 2001 model.

## Training pipeline

### 1. Change a directory in your terminal to object_detection.

```bash
cd models/object_detection
```
If You have not created virtual environment yet:
```bash
./init_venv.sh
```
Activate virtual environment:
```bash
source venv/bin/activate
```

### 2. Select a model template file and instantiate it in some directory.

```bash
export MODEL_TEMPLATE=`realpath ./model_templates/person-vehicle-bike-detection/person-vehicle-bike-detection-2000/template.yaml`
export WORK_DIR=/tmp/my_model
python ../../tools/instantiate_template.py ${MODEL_TEMPLATE} ${WORK_DIR}
```

### 3. Collect dataset

Collect or download images with `vehicle`, `person`, `non-vehicle` objects presented on them.

### 4. Prepare annotation

Annotate dataset and save annotation to MSCOCO format with `vehicle`, `person`, `non-vehicle` as classes or you can start with existing toy data.

```bash
export OBJ_DET_DIR=`pwd`
export TRAIN_ANN_FILE="${OBJ_DET_DIR}/../../data/airport/annotation_example_train.json"
export TRAIN_IMG_ROOT="${OBJ_DET_DIR}/../../data/airport/train"
export VAL_ANN_FILE="${OBJ_DET_DIR}/../../data/airport/annotation_example_val.json"
export VAL_IMG_ROOT="${OBJ_DET_DIR}/../../data/airport/val"
```

### 5. Change current directory to directory where the model template has been instantiated.

```bash
cd ${WORK_DIR}
```

### 6. Training and Fine-tuning

Try both following variants and select the best one:

   * **Training** from scratch or pre-trained weights. Only if you have a lot of data, let's say tens of thousands or even more images. This variant assumes long training process starting from big values of learning rate and eventually decreasing it according to a training schedule.
   * **Fine-tuning** from pre-trained weights. If the dataset is not big enough, then the model tends to overfit quickly, forgetting about the data that was used for pre-training and reducing the generalization ability of the final model. Hence, small starting learning rate and short training schedule are recommended.

   * If you would like to start **training** from pre-trained weights use `--load-weights` pararmeter.

      ```bash
      python train.py \
         --load-weights ${WORK_DIR}/snapshot.pth \
         --train-ann-files ${TRAIN_ANN_FILE} \
         --train-data-roots ${TRAIN_IMG_ROOT} \
         --val-ann-files ${VAL_ANN_FILE} \
         --val-data-roots ${VAL_IMG_ROOT} \
         --save-checkpoints-to ${WORK_DIR}/outputs
      ```

      Also you can use parameters such as `--epochs`, `--batch-size`, `--gpu-num`, `--base-learning-rate`, otherwise default values will be loaded from `${MODEL_TEMPLATE}`.

   * If you would like to start **fine-tuning** from pre-trained weights use `--resume-from` parameter and value of `--epochs` have to exceed the value stored inside `${MODEL_TEMPLATE}` file, otherwise training will be ended immediately. Here we add `5` additional epochs.

      ```bash
      export ADD_EPOCHS=5
      export EPOCHS_NUM=$((`cat ${MODEL_TEMPLATE} | grep epochs | tr -dc '0-9'` + ${ADD_EPOCHS}))

      python train.py \
         --resume-from ${WORK_DIR}/snapshot.pth \
         --train-ann-files ${TRAIN_ANN_FILE} \
         --train-data-roots ${TRAIN_IMG_ROOT} \
         --val-ann-files ${VAL_ANN_FILE} \
         --val-data-roots ${VAL_IMG_ROOT} \
         --save-checkpoints-to ${WORK_DIR}/outputs \
         --epochs ${EPOCHS_NUM}
      ```

### 7. Evaluation

Evaluation procedure allows us to get quality metrics values and complexity numbers such as number of parameters and FLOPs.

To compute MS-COCO metrics and save computed values to `${WORK_DIR}/metrics.yaml` run:

```bash
python eval.py \
   --load-weights ${WORK_DIR}/outputs/latest.pth \
   --test-ann-files ${VAL_ANN_FILE} \
   --test-data-roots ${VAL_IMG_ROOT} \
   --save-metrics-to ${WORK_DIR}/metrics.yaml
```

You can also save images with predicted bounding boxes using `--save-output-to` parameter.

```bash
python eval.py \
   --load-weights ${WORK_DIR}/outputs/latest.pth \
   --test-ann-files ${VAL_ANN_FILE} \
   --test-data-roots ${VAL_IMG_ROOT} \
   --save-metrics-to ${WORK_DIR}/metrics.yaml \
   --save-output-to ${WORK_DIR}/output_images
```

### 8. Export PyTorch\* model to the OpenVINO™ format

To convert PyTorch\* model to the OpenVINO™ IR format run the `export.py` script:

```bash
python export.py \
   --load-weights ${WORK_DIR}/outputs/latest.pth \
   --save-model-to ${WORK_DIR}/export
```

This produces model `model.xml` and weights `model.bin` in single-precision floating-point format
(FP32). The obtained model expects **normalized image** in planar BGR format.

For SSD networks an alternative OpenVINO™ representation is saved automatically to `${WORK_DIR}/export/alt_ssd_export` folder.
SSD model exported in such way will produce a bit different results (non-significant in most cases),
but it also might be faster than the default one. As a rule SSD models in [Open Model Zoo](https://github.com/opencv/open_model_zoo/) are exported using this option.

### 9. Validation of IR

Instead of passing `snapshot.pth` you need to pass path to `model.bin`.

```bash
python eval.py \
   --load-weights ${WORK_DIR}/export/model.bin \
   --test-ann-files ${VAL_ANN_FILE} \
   --test-data-roots ${VAL_IMG_ROOT} \
   --save-metrics-to ${WORK_DIR}/metrics.yaml
```
