### Depth Coefficients for Depth Completion
**Saif Imran** **Yunfei Long** **Xiaoming Liu** **Daniel Morris**

The site contains Tensorflow implementation of our work "Depth Coefficients for Depth Completion" featured in CVPR 2019. More description at the link to the [paper.](https://arxiv.org/abs/1903.05421)

## Overview

The following gives a overall illustration of our work.  
![Image](/images/overview_cropped.png)

# Implementation Details
Our system comprises of 32Gb RAM, a 1080 Ti GPU Card and Ubuntu 16.10 as OS. Due to limited GPU memory, we needed to train the KITTI data with patches compared to full sized images of KITTI. 
# Prerequisities
The codebase was developed and tested in Ubuntu 16.10, Tensorflow 1.10, python 3.5, CUDA 9.0 and CuDNN 7.0. Please see the tensorflow [installation] (https://www.tensorflow.org/install/pip) for details. Make sure you have the right versions of Cuda 9.0 and CudNN available. Then steps are:

1. Create conda environment by `conda create -n tensorflow1.10_py3.5 python=3.5`

2. Activate the environment by `source activate tensorflow1.10_py3.5`

3. Install the tensorflow-gpu version by `pip install --ignore-installed --upgrade https://storage.googleapis.com/tensorflow/linux/gpu/tensorflow_gpu-1.10.1-cp35-cp35m-linux_x86_64.whl`

4. Pip install the required libraries: scipy, imageio, matplotlib, easydict, sklearn, scikit-image

# Dataset Generation
We use the [KITTI depth completion dataset](http://www.cvlibs.net/datasets/kitti/eval_depth.php?benchmark=depth_completion) for training our network. KITTI depth completion dataset provides Velodyne HDL-64E as raw lidar scans as input and provide semi-dense annotated ground-truth data for training (see the paper [Sparsity Invariant CNNs](https://arxiv.org/abs/1708.06500)). But what makes our case interesting is how we subsample the 64R raw lidar scans to make it 32R, 16R lidar-scans respectively. We needed to split the lidar rows based on azimuth angle in lidar space (see our paper), so we required [KITTI raw dataset](http://www.cvlibs.net/datasets/kitti/raw_data.php) to access the raw lidar scans, skip the desired number of rows and then project the lidar scans in the image plane. We provide a sample matlab code that can do the data-mapping between KITTI's depth completion and raw dataset and generate the subsampled data which is used for training eventually. So the steps to prepare the subsampled data are:

1. Download the KITTI depth completion dataset, importantly the annotated ground-truth data, and the manually selected validation and test dataset.
2. Download the KITTI raw dataset. Create 'KITTI_Raw' directory in ./Data/ and copy the raw_data_downloader script provided in that folder. For downloading all raw data from the KITTI websites, then cd to the 'KITTI_Raw' directory run the .sh script from the command line:

./raw_data_downloader.sh

It will download the zip files and extract them into a coherent data structure: Each folder contains all sequences recorded at a single day, including the calibration files for that day.

3. Instead of using velodyne_raw as provided by KITTI Depth Completion data, prepare subsampled data as input to the network. The subsampled data typically comprises of 0x0_nSKips (64R Lidar Scans), 1x0_nSKips (32R Lidar Scans), 4x0_nSKips (16R Lidar Scans) respectively. Run the *trainval_origpadgenerator_subsample.m* in ./Codes/Matlab_Scripts. You might need to change the absolute path in the provided scripts. You can prepare the subsampled data for both 'train' and 'val' set using the script just by changing the `dataset_type` to `train` or `val` in the script.

The validation shortened dataset are the manually selected validation set provided by KITTI. To generate the subsampled data of selected validation set, please use the *generate_valshortened.m* script.  

The overall directory structure should look the following:
```
.
Depth-Coefficients-from-Single-Image
| Data
  |__KITTI_Depth
     |__train
        |__2011_09_26_drive_0001_sync
            |__color
                |__image_02
                    0000000005.png
                |__image_03
            |__proj_depth
                |__groundtruth
                    |__image_02
                        0000000005.png
                |__0x0_nSKips
                    |__image_02
                        0000000005.png
                    |__image_03
                |__4x0_nSKips
                    |__image_02
                        0000000005.png
                    |__image_03
                |__...
        |__...
     |__val
        |__2011_09_26_drive_0002_sync            
        |__...
     |__val_shortened
        |__image
            2011_09_26_drive_0002_sync_image_02_0000000005.png
        |__proj_depth
            |__0x0_nSKips
                2011_09_26_drive_0002_sync_image_02_0000000005.png
            |__4x0_nSKips
                2011_09_26_drive_0002_sync_image_02_0000000005.png
            |__groundtruth
                2011_09_26_drive_0002_sync_image_02_0000000005.png
      |__val_selection_cropped                
  |__KITTI_Raw
      |__2011_09_26
          |__2011_09_26_001_sync
              |__...
              |__image_02
              |__image_03
              |__oxts
              |__velodyne_points
          |__...
      |__...          
```

# Network
We use the following configuration to train the network. In this implementation, we used a Resnet-18 network and batch-size of 3 since the model can be fit easily in a single GPU. However in the paper, we used Resnet-34, and we found the bigger network to improve performance by 2-3 cms. Note that in this implementation, we also found that by adding a simple auxiliary loss at the encoder network, the performance improves compared to the reported performance in the paper. So we suggest the readers to stick to the new training strategy when training the network. 
![Image](/images/DC_Network.png)

# Training
For training the network, run the  train_resnet.py script by `python train_resnet.py`. You can change the name of the model, the DC channels, patch-size of the KITTI data etc in the script. All the models will be saved in `checkpoint_dir`. The training typically takes 24-30 hours for Resnet-18 network and 40-48 hours for Resnet-34 network.

# Pretrained Models
We provide the Resnet-18 model with DC channels 80 for 16R Subsampling or KITTI data. We will upload the Resnet-34 model shortly. Download the [Resnet18 model](https://drive.google.com/drive/folders/1ul_-yiFExztcVTCkjm-YHmLaaxx77ThA?usp=sharing) in `checkpoint_dir` directory.
# Evaluation
We also provide an evaluation script to evaluate the model. To evaluate the model, just copy and paste the name of the model in the evaluation_script and run `python evaluate_resnet.py`. It will evaluate the model based the metrics MAE, RMSE, TMAE, TRMSE etc. The evaluation can be done on two options: DC using 3 coeffs, and DC using all coeffs. 

To evaluate on the DC using 3 coeffs, type `python evaluate_resnet.py --coeftype 3coef`. To evaluate it on all coeffs, type `python evaluate_resnet.py --coeftype allcoef`. 

Right now, we evaluate it by tiling up all the patches of size 352x420 to make it full sized 352x1242 image. To handle the overlapping between patches, we just sum the overlapped DC vectors and normalize it to make sure the sum of weights are not greater than 1. To evaluate it on full-sized image, change the patch-size to 352 x 1242 in the KITTI_Params of the evaluation script and then run the script using: `python evaluate_resnet.py --splitFlag F`

Evaluating it on full-sized image has slighly worse performance compared to patch-sized image.  

## Video Demo
Here is a video demonstration of the work in a typical KITTI sequence:
![DC_Video](/images/DC.gif)
[Youtube Video](https://www.youtube.com/watch?v=ghDFX2hQbYY)



## Citations
If you use our method and code in your work, please cite the following:

@inproceedings{ depth-coefficients-for-depth-completion, 
  author = { Saif Imran and Yunfei Long and Xiaoming Liu and Daniel Morris },
  title = { Depth Coefficients for Depth Completion },
  booktitle = { In Proceeding of IEEE Computer Vision and Pattern Recognition },
  address = { Long Beach, CA },
  month = { January },
  year = { 2019 },
}
