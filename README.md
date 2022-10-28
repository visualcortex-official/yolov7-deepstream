# How to run YOLOv7 on NVIDIA Deepstream

Instructions for executing the official [YOLOv7](https://github.com/WongKinYiu/yolov7) on NVIDIA Deepstream.

## 1. Export the Model

The [YOLOv7](https://github.com/WongKinYiu/yolov7) already has a method for exporting a trained model to an [ONNX](https://onnx.ai/) format which can be run like this:

```
python export.py \
--weights ./yolov7-tiny.pt \
--grid \
--end2end \
--simplify \
--topk-all 100 \
--iou-thres 0.65 \
--conf-thres 0.35 \
--img-size 640 640
```

This command will create an ONNX model with an [efficientNMS](https://github.com/NVIDIA/TensorRT/tree/main/plugin/efficientNMSPlugin) (Non Max Supression) node inserted. Care should be taken to ensure the parameters `topk-all`, `iou-thres` and `conf-thres` are configured correctly for how you want to execute your model. See the [efficientNMS](https://github.com/NVIDIA/TensorRT/tree/main/plugin/efficientNMSPlugin) README for further details.

## 2. Update the Custom Bounding Box Parser

At time of writing, NVIDIA does not support parsing the `efficientNMS` output format into the correct data structure for Deepstream [NvDsObjectMeta](https://docs.nvidia.com/metropolis/deepstream/sdk-api/struct__NvDsObjectMeta.html). We are in progress of donating this code to NVIDIA so it can be included in the Deepstream releases by default so this step may not be required in future.

To be able to map the model outputs from `efficientNMS` to the NVIDIA `NvDsObjectMeta` data structure you need this code has been added in `/opt/nvidia/deepstream/deepstream-6.1/sources/libs/nvdsinfer_customparser/nvdsinfer_custombboxparser.cpp`. A copy of this modified file exists in this repository that you should use to replace the file in that directory.

Once replaced, run `make` in the `/opt/nvidia/deepstream/deepstream-6.1/sources/libs/nvdsinfer_customparser` directory to compile the library with the new function.

## 3. Update your nvinfer Configuration

You then just need to make sure your [nvinfer](https://docs.nvidia.com/metropolis/deepstream/dev-guide/text/DS_plugin_gst-nvinfer.html) stages point to the correct custom library and function like:

```ini
parse-bbox-func-name=NvDsInferParseCustomEfficientNMS
custom-lib-path=/opt/nvidia/deepstream/deepstream-6.1/sources/libs/nvdsinfer_customparser/libnvds_infercustomparser.so
```

# Common Problems

## numClassesConfigured Assertion

Messages like this indicate that at least one of the detections had a class id larger than the `nvinfer` configured `num-detected-classes`. The solution is to update `num-detected-classes` to the correct number of classes for your model.

```
Assertion `(unsigned int) p_classes[i] < detectionParams.numClassesConfigured’ failed.
```

# License

The original `/opt/nvidia/deepstream/deepstream-6.1/sources/libs/nvdsinfer_customparser/nvdsinfer_custombboxparser.cpp` contains this license which this code retains.

Changes by VisualCortex are under the `MIT License` and a `LICENSE` file is attached.

```
Copyright (c) 2018-2020, NVIDIA CORPORATION. All rights reserved.

Permission is hereby granted, free of charge, to any person obtaining a
copy of this software and associated documentation files (the "Software"),
to deal in the Software without restriction, including without limitation
the rights to use, copy, modify, merge, publish, distribute, sublicense,
and/or sell copies of the Software, and to permit persons to whom the
Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.  IN NO EVENT SHALL
THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
DEALINGS IN THE SOFTWARE.
```
