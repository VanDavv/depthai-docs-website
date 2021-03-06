---
layout: default
title: Tutorial - How to use a Pre-trained OpenVINO model on DepthAI
toc_title: Pre-trained OpenVINO model
description: Learn how to detect faces in realtime - even on a low-powered Raspberry Pi - with a pre-trained model.
og_image_path: "/images/tutorials/pretrained_model/previewout.png"
order: 3
---

# {{ page.title }}

In this tutorial, you'll learn how to detect faces in realtime, even on a low-powered Raspberry Pi. I'll introduce you to the OpenVINO toolset, the Open Model Zoo (where we'll download the [face-detection-retail-0004](https://github.com/opencv/open_model_zoo/blob/2019_R2/intel_models/face-detection-retail-0004/description/face-detection-retail-0004.md) model), and show you how to generate the files needed to run model inference on your DepthAI board.

![model image](/images/tutorials/pretrained_model/previewout2.png)

Haven't heard of OpenVINO or the Open Model Zoo? I'll start with a quick introduction of why we need these tools.

## What is OpenVINO?

Under-the-hood, DepthAI uses the Intel MyriadX chip to perform high-speed model inference. However, you can't just dump your neural net into the chip and get high-performance for free. That's where [OpenVINO](https://docs.openvinotoolkit.org/) comes in. OpenVINO is a free toolkit that converts a deep learning model into a format that runs on Intel Hardware. Once the model is converted, it's common to see Frames Per Second (FPS) improve by 25x or more. Are a couple of small steps worth a 25x FPS increase? Often, the answer is yes!

## What is the Open Model Zoo?

The [Open Model Zoo](https://github.com/opencv/open_model_zoo) is a library of freely-available pre-trained models. The Zoo also contains scripts for downloading those models into a compile-ready format to run on DepthAI.

DepthAI is able to run many of the object detection models in the Zoo.

## Dependencies

This tutorial has the same dependencies as the [Hello World Tutorial](/tutorials/hello_world#dependencies) with one addition: OpenVINO version `2019_R2`. We'll verify your OpenVINO install below.

## Verify your OpenVINO install

<div class="alert alert-primary" role="alert">
<i class="material-icons">
error
</i>
  Using the RPi Compute edition or a pre-flashed DepthAI µSD card? <strong>Skip this step.</strong><br/>
  <span class="small">OpenVINO version `2019_R2` has been pre-installed.</span>
</div>

DepthAI requires OpenVINO version `2019_R2`. Let's verify that this version is installed on your host. Check your version by running the following from a terminal session:

```
cat /opt/intel/openvino/inference_engine/version.txt
```

You should see output similar to:

```
Thu Jul 18 19:41:35 MSK 2019
f5827d4773ebbe727c9acac5f007f7d94dd4be4e
releases_2019_R2_InferenceEngine_27579
```

Verify that you see `releases_2019_R2` in your output. If you do, move on. If you are on a different version, goto the [OpenVINO site](https://docs.openvinotoolkit.org/2019_R2/index.html) and download the `2019_R2` version for your OS:

![openvino_version](/images/openvino_version.png)

{: data-toc-title="Model Downloader"}
## Check if the Model Downloader is installed

When installing OpenVINO, you can choose to perform a smaller install to save disk space. This custom install may not include the model downloader script. Lets check if the downloader was installed. In a terminal session, type the following:

```
find /opt/intel/ -iname downloader.py
find ~ -iname downloader.py
```

__Move on if you see the output below__:

```
/opt/intel/openvino_2019.2.242/deployment_tools/open_model_zoo/tools/downloader/downloader.py
```

__Didn't see any output?__ Don't fret if `downloader.py` isn't found. We'll install this below.

{: data-toc-title="Install"}
### Install Open Model Zoo Downloader

If the downloader tools weren't found, we'll install the tools by cloning the [Open Model Zoo Repo](https://github.com/opencv/open_model_zoo/blob/2019_R2/tools/downloader/README.md) and installing the tool dependencies.

Start a terminal session and run the following commands in your terminal:

```
cd ~
git clone https://github.com/opencv/open_model_zoo.git
cd open_model_zoo
git checkout tags/2019_R2
cd tools/downloader
python3 -m pip install --user -r ./requirements.in
```

This clones the repo into a `~/open_model_zoo` directory, checks out the required `2019_R2` version, and installs the downloader dependencies.

{: data-toc-title="Setup Downloader env var."}
## Create an OPEN_MODEL_DOWNLOADER environment variable

Typing the full path to `downloader.py` can use a lot of keystrokes. In an effort to extend your keyboard life, let's store the path to this script in an environment variable.

Run the following in your terminal:

```
OPEN_MODEL_DOWNLOADER='INSERT PATH TO YOUR downloader.py SCRIPT'
```

Where `INSERT PATH TO YOUR downloader.py SCRIPT` can be found via:

```
find /opt/intel/ -iname downloader.py
find ~ -iname downloader.py
```

For example, if you are using the RPi Compute and installed `open_model_zoo` yourself:

```
OPEN_MODEL_DOWNLOADER='/home/pi/open_model_zoo/tools/downloader/downloader.py'
```

Verify this was set correctly:

```
echo $OPEN_MODEL_DOWNLOADER
/home/pi/open_model_zoo/tools/downloader/downloader.py
```

{: data-toc-title="Download the model"}
## Download the face-detection-retail-0004 model

We've installed everything we need to download models from the Open Model Zoo! We'll now use the [Model Downloader](https://github.com/opencv/open_model_zoo/blob/2019_R2/tools/downloader/README.md) to download the `face-detection-retail-0004` model files. Run the following in your terminal:

```
$OPEN_MODEL_DOWNLOADER --name face-detection-retail-0004 --output_dir ~/open_model_zoo_downloads/
```

This will download the model files to `~/open_model_zoo_downloads/`. Specifically, the model files we need are located at:

```
~/open_model_zoo_downloads/Retail/object_detection/face/sqnet1.0modif-ssd/0004/dldt/FP16
```

You'll see two files within the directory:

```
ls -lh ~/open_model_zoo_downloads/Retail/object_detection/face/sqnet1.0modif-ssd/0004/dldt/FP16/
total 1.2M
-rw-r--r-- 1 pi pi 1.2M Feb 28 10:04 face-detection-retail-0004.bin
-rw-r--r-- 1 pi pi  47K Feb 28 10:04 face-detection-retail-0004.xml
```

The model is in the OpenVINO Intermediate Representation (IR) format:

* face-detection-retail-0004.xml - Describes the network topology
* face-detection-retail-0004.bin - Contains the weights and biases binary data.

This means we are ready to compile the model for the MyriadX!

## Compile the model

The MyriadX chip used on our DepthAI board does not use the IR format files directly. Instead, we need to generate two files:

* `face-detection-retail-0004.blob` - We'll create this file with the `myriad_compile` command.
* `face-detection-retail-0004.json` - A `blob_file_config` file in JSON format. This describes the format of the output tensors.

We'll start by creating the `blob` file.

### Locate myriad_compile

Let's find where `myriad_compile` is located. In your terminal, run:

```
find /opt/intel/ -iname myriad_compile
```

On an RPi compute, I see the following:

```
find /opt/intel/ -iname myriad_compile
/opt/intel/openvino/deployment_tools/inference_engine/lib/armv7l/myriad_compile
```

On an Intel-based architecture, `armv7l` will be `intel64`.

Since it's such a long path, let's store the `myriad_compile` executable in an environment variable (just like `OPEN_MODEL_DOWNLOADER`). On an RPi:

```
MYRIAD_COMPILE='/opt/intel/openvino/deployment_tools/inference_engine/lib/armv7l/myriad_compile'
```

### Run myriad_compile

```
$MYRIAD_COMPILE -m ~/open_model_zoo_downloads/Retail/object_detection/face/sqnet1.0modif-ssd/0004/dldt/FP16/face-detection-retail-0004.xml -ip U8 -VPU_MYRIAD_PLATFORM VPU_MYRIAD_2480 -VPU_NUMBER_OF_SHAVES 4 -VPU_NUMBER_OF_CMX_SLICES 4
```

You should see:

```
Inference Engine:
	API version ............ 2.0
	Build .................. custom_releases/2019/R2_f5827d4773ebbe727c9acac5f007f7d94dd4be4e
	Description ....... API
Done
```

Where's the blob file? It's located in the same folder as `face-detection-retail-0004.xml`:

```
ls -lh ~/open_model_zoo_downloads/Retail/object_detection/face/sqnet1.0modif-ssd/0004/dldt/FP16
total 2.5M
-rw-r--r-- 1 pi pi 1.2M Feb 28 10:04 face-detection-retail-0004.bin
-rw-r--r-- 1 pi pi 1.3M Feb 28 10:37 face-detection-retail-0004.blob
-rw-r--r-- 1 pi pi  47K Feb 28 10:04 face-detection-retail-0004.xml
```

{: #blob_file_config}
## Create the blob config file

The MyriadX needs both a `blob` file (which we just created) and a `blob_file_config` in JSON format. We'll create this config file manually. In your terminal:

```
cd ~/open_model_zoo_downloads/Retail/object_detection/face/sqnet1.0modif-ssd/0004/dldt/FP16
touch face-detection-retail-0004.json
```

Copy and paste the following into `face-detection-retail-0004.json`:

```json
{
    "tensors":
    [
        {
            "output_tensor_name": "out",
            "output_dimensions": [1, 1, 100, 7],
            "output_entry_iteration_index": 2,
            "output_properties_dimensions": [3],
            "property_key_mapping":
            [
                [],
                [],
                [],
                ["image_id", "label", "conf", "x_min", "y_min", "x_max", "y_max"]
            ],
            "output_properties_type": "f16"
        }
    ]
}
```

What do these values mean?

* `output_dimensions` - The neural net has an [output format](https://docs.openvinotoolkit.org/latest/_models_intel_face_detection_retail_0004_description_face_detection_retail_0004.html#outputs) of `[1, 1, N, 7]` where `N` is the number of detected bounding boxes. Each detection has the format: `[image_id, label, conf, x_min, y_min, x_max, y_max]`. We specify `100` instead of `N` to indicate we'll handle up to 100 bounding boxes, which should be more than enough possible faces to detect.
* `output_entry_iteration_index` - Indicates the array position we'll use to iterate over detections. `N` is in the 2nd position (index start = 0).
* `property_key_mapping` - Maps the properties to friendly names we can access when running model inference in our too-be-created Python script.
* `output_properties_type` - We're using a model with `f16` precision.

{: data-toc-title="Run the model"}
## Run and display the model output

With both `blob` and a `json` blob config file, we're ready to roll! We'll create a Python script to display the results of our face detections in realtime.

### Setup the Python script directory
If you haven't created a `{{site.tutorials_dir}}` folder yet, lets do that:

```
cd ~
mkdir -p {{site.tutorials_dir}}/2-face-detection-retail
cd {{site.tutorials_dir}}/2-face-detection-retail
```

### Install pip dependencies

We need to import a small number of packages to run our Python script. Download and install the requirements for this tutorial:

```
wget https://raw.githubusercontent.com/luxonis/depthai-tutorials/master/2-face-detection-retail/requirements.txt
pip3 install --user -r requirements.txt
```

### Download our script

Download [this ready-to-go script](https://github.com/luxonis/depthai-tutorials/blob/master/2-face-detection-retail/face-detection-retail-0004.py) for running the `face-detection-retail-0004` model:

```
wget https://raw.githubusercontent.com/luxonis/depthai-tutorials/master/2-face-detection-retail/face-detection-retail-0004.py
```

The script assumes the `blob` and `json` files are located in the directory we used in this tutorial:

```
~/open_model_zoo_downloads/Retail/object_detection/face/sqnet1.0modif-ssd/0004/dldt/FP16`
```

Execute the script to see an annotated video stream of face detections:

```
python3 face-detection-retail-0004.py
```

You should see output annotated output similar to:

![model image](/images/tutorials/pretrained_model/previewout.png)

Substitute your face for mine, of course.

## Reviewing the flow

The flow we walked through works for other pre-trained object detection models in the Open Model Zoo:

1. Download the model:
    ```
    $OPEN_MODEL_DOWNLOADER --name [INSERT MODEL NAME] --output_dir ~/open_model_zoo_downloads/
    ```
2. Create the MyriadX blob file:
    ```
    $MYRIAD_COMPILE -m [INSERT PATH TO MODEL XML FILE] -ip U8 -VPU_MYRIAD_PLATFORM VPU_MYRIAD_2480 -VPU_NUMBER_OF_SHAVES 4 -VPU_NUMBER_OF_CMX_SLICES 4
    ```
3. Create a [JSON config file](#blob_file_config) based on the model output.
4. Write a script (similar [to this](https://github.com/luxonis/depthai-tutorials/blob/master/2-face-detection-retail/face-detection-retail-0004.py)) utilizing the DepthAI API, specifying the path to the blob file and its config via the `blob_file` and `blob_file_config` Pipeline config settings.
