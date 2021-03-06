# Faster-RCNN / Mask-RCNN on COCO
This example provides a minimal (<2k lines) and faithful implementation of the following papers:

+ [Faster R-CNN: Towards Real-Time Object Detection with Region Proposal Networks](https://arxiv.org/abs/1506.01497)
+ [Feature Pyramid Networks for Object Detection](https://arxiv.org/abs/1612.03144)
+ [Mask R-CNN](https://arxiv.org/abs/1703.06870)

with the support of:
+ Multi-GPU / distributed training
+ [Cross-GPU BatchNorm](https://arxiv.org/abs/1711.07240)
+ [Group Normalization](https://arxiv.org/abs/1803.08494)

## Dependencies
+ Python 3; OpenCV.
+ TensorFlow >= 1.6 (1.4 or 1.5 can run but may crash due to a TF bug);
+ pycocotools: `pip install 'git+https://github.com/cocodataset/cocoapi.git#subdirectory=PythonAPI'`
+ Pre-trained [ImageNet ResNet model](http://models.tensorpack.com/FasterRCNN/)
  from tensorpack model zoo.
+ COCO data. It needs to have the following directory structure:
```
COCO/DIR/
  annotations/
    instances_train2014.json
    instances_val2014.json
    instances_minival2014.json
    instances_valminusminival2014.json
  train2014/
    COCO_train2014_*.jpg
  val2014/
    COCO_val2014_*.jpg
```
`minival` and `valminusminival` are optional. You can download them
[here](https://github.com/rbgirshick/py-faster-rcnn/blob/master/data/README.md).


## Usage
### Train:

On a single machine:
```
./train.py --config \
    MODE_MASK=True MODE_FPN=True \
    DATA.BASEDIR=/path/to/COCO/DIR \
    BACKBONE.WEIGHTS=/path/to/ImageNet-R50-Pad.npz \
```

To run distributed training, set `TRAINER=horovod` and refer to [HorovodTrainer docs](http://tensorpack.readthedocs.io/modules/train.html#tensorpack.train.HorovodTrainer).

Options can be changed by either the command line or the `config.py` file.
Recommended configurations are listed in the table below.

The code is only valid for training with 1, 2, 4 or >=8 GPUs.
Not training with 8 GPUs may result in different performance from the table below.

To predict on an image (and show output in a window):
```
./train.py --predict input.jpg --load /path/to/model --config SAME-AS-TRAINING
```

Evaluate the performance of a model on COCO.
(Several trained models can be downloaded in [model zoo](http://models.tensorpack.com/FasterRCNN)):
```
./train.py --evaluate output.json --load /path/to/COCO-R50C4-MaskRCNN-Standard.npz \
    --config MODE_MASK=True DATA.BASEDIR=/path/to/COCO/DIR
```
Evaluation or prediction will need the same `--config` used during training.

## Results

These models are trained with different configurations on trainval35k and evaluated on minival using mAP@IoU=0.50:0.95.
Performance in [Detectron](https://github.com/facebookresearch/Detectron/) can be roughly reproduced, some are better but some are worse.
MaskRCNN results contain both box and mask mAP.

 | Backbone | mAP<br/>(box;mask)                                                                                       | Detectron mAP <sup>[1](#ft1)</sup><br/> (box;mask) | Time on 8 V100s | Configurations <br/> (click to expand)                                                                                                                                                                          |
 | -        | -                                                                                                        | -                                                  | -               | -                                                                                                                                                                                                               |
 | R50-C4   | 33.1                                                                                                     |                                                    | 18h             | <details><summary>super quick</summary>`MODE_MASK=False FRCNN.BATCH_PER_IM=64`<br/>`PREPROC.SHORT_EDGE_SIZE=600 PREPROC.MAX_SIZE=1024`<br/>`TRAIN.LR_SCHEDULE=[150000,230000,280000]` </details>                |
 | R50-C4   | 36.6                                                                                                     | 36.5                                               | 44h             | <details><summary>standard</summary>`MODE_MASK=False` </details>                                                                                                                                                |
 | R50-FPN  | 37.4                                                                                                     | 37.9                                               | 30h             | <details><summary>standard</summary>`MODE_MASK=False MODE_FPN=True` </details>                                                                                                                                  |
 | R50-C4   | 37.8;33.1 [:arrow_down:](http://models.tensorpack.com/FasterRCNN/COCO-R50C4-MaskRCNN-Standard.npz)       | 37.8;32.8                                          | 49h             | <details><summary>standard</summary>`MODE_MASK=True` </details>                                                                                                                                                 |
 | R50-FPN  | 38.2;34.9 [:arrow_down:](http://models.tensorpack.com/FasterRCNN/COCO-R50FPN-MaskRCNN-Standard.npz)      | 38.6;34.5                                          | 32h             | <details><summary>standard</summary>`MODE_MASK=True MODE_FPN=True` </details>                                                                                                                                   |
 | R50-FPN  | 39.1;35.2                                                                                                | 38.6;34.5                                          | 32h             | <details><summary>better params</summary>`MODE_MASK=True MODE_FPN=True`<br/>`TEST.RESULT_SCORE_THRESH=1e-4`<br/>`FRCNN.BBOX_REG_WEIGHTS=[20,20,10,10]` </details>                                               |
 | R50-FPN  | 39.5;35.2                                                                                                | 39.5;34.4<sup>[2](#ft2)</sup>                      | 34h             | <details><summary>standard+ConvGNHead</summary>`MODE_MASK=True MODE_FPN=True`<br/>`FPN.FRCNN_HEAD_FUNC=fastrcnn_4conv1fc_gn_head` </details>                                                                    |
 | R50-FPN  | 40.0;36.1 [:arrow_down:](http://models.tensorpack.com/FasterRCNN/COCO-R50FPN-MaskRCNN-StandardGN.npz)    | 40.3;35.7                                          | 44h             | <details><summary>standard+GN</summary>`MODE_MASK=True MODE_FPN=True`<br/>`FPN.NORM=GN BACKBONE.NORM=GN`<br/>`FPN.FRCNN_HEAD_FUNC=fastrcnn_4conv1fc_gn_head`<br/>`FPN.MRCNN_HEAD_FUNC=maskrcnn_up4conv_gn_head` |
 | R101-C4  | 40.8;35.1 [:arrow_down:](http://models.tensorpack.com/FasterRCNN/COCO-R101C4-MaskRCNN-Standard.npz)      |                                                    | 63h             | <details><summary>standard</summary>`MODE_MASK=True `<br/>`BACKBONE.RESNET_NUM_BLOCK=[3,4,23,3]` </details>                                                                                                     |
 | R101-FPN | 40.4;36.6 [:arrow_down:](http://models.tensorpack.com/FasterRCNN/COCO-R101FPN-MaskRCNN-Standard.npz)     | 40.9;36.4                                          | 40h             | <details><summary>standard</summary>`MODE_MASK=True MODE_FPN=True`<br/>`BACKBONE.RESNET_NUM_BLOCK=[3,4,23,3]` </details>                                                                                        |
 | R101-FPN | 41.0;36.6 [:arrow_down:](http://models.tensorpack.com/FasterRCNN/COCO-R101FPN-MaskRCNN-BetterParams.npz) | 40.9;36.4                                          | 40h             | <details><summary>better params</summary>`MODE_MASK=True MODE_FPN=True`<br/>`BACKBONE.RESNET_NUM_BLOCK=[3,4,23,3]`<br/>`TEST.RESULT_SCORE_THRESH=1e-4`<br/>`FRCNN.BBOX_REG_WEIGHTS=[20,20,10,10]` </details>    |

 <a id="ft1">1</a>: Here we comapre models that have identical training & inference cost between the two implementation. However their numbers are different due to many small implementation details.

 <a id="ft2">2</a>: Numbers taken from [Group Normalization](https://arxiv.org/abs/1803.08494)

## Notes

See [Notes on This Implementation](NOTES.md)
