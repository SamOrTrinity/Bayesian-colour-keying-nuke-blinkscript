# Bayesian Colour Keying in Nuke using BlinkScript

This repository contains an implementation and evaluation of a Bayesian colour keying pipeline for green-screen foreground extraction. The project was completed for the 5C37 Motion Picture Engineering module and explores how probabilistic modelling, Markov Random Field smoothing and Nuke BlinkScript can be used to estimate an alpha matte from green-screen footage.

The project implements a classical Bayesian keyer and compares its output against a provided U-Net matte and a Keylight matte using quantitative and visual evaluation.

## Project Summary

The objective of the project is to separate the foreground subject from a green-screen background by estimating a binary alpha matte.

The observed image is modelled using the compositing equation

```text
C_p = alpha_p  f_p + (1 - alpha_p)  b + epsilon
```

where

 `C_p` is the observed pixel colour.
 `alpha_p` is the unknown matte label.
 `f_p` is the foreground colour.
 `b` is the background colour model.
 `epsilon` is a Gaussian noise term.

The problem is formulated as a maximum a posteriori (MAP) estimation problem. The final matte is obtained by combining

1. A colour-based Bayesian data likelihood.
2. A spatial smoothness prior using a Markov Random Field.
3. Iterative local optimisation using Iterated Conditional Modes.

## Main Features

 Bayesian green-screen colour keying.
 Nuke BlinkScript implementation.
 Background modelling in YCbCr colour space.
 Gaussian background likelihood using mean and variance from a selected background crop.
 Weighted Mahalanobis-style data cost.
 Binary matte initialisation from data-cost thresholding.
 Markov Random Field prior over an 8-neighbourhood.
 Iterative matte refinement using repeated MRF update nodes.
 Quantitative comparison using MSE and SSIM.
 Visual comparison against U-Net and Keylight outputs.
 Composite reconstruction using the estimated alpha matte.

## Methodology

### 1. Colour Space Conversion

The input image is converted from RGB to YCbCr. YCbCr was chosen because it separates luminance from chrominance, making the background model more robust to lighting changes. In green-screen keying, chroma channels are more reliable for identifying the screen colour than raw RGB intensity.

The luma channel is given a smaller weight during data-cost computation so that the keyer is less sensitive to brightness variation and more sensitive to chromatic deviation from the green screen.

### 2. Background Model Estimation

A clean background region is cropped from the YCbCr image. This crop is assumed to contain only green-screen background pixels.

From this crop, the pipeline computes

 Per-channel background mean.
 Per-channel background variance.

The reduction is performed using a custom BlinkScript reduction kernel, repeatedly reducing the crop until a 1×1 mean or variance estimate is obtained.

### 3. Data Cost Computation

For every pixel, the background energy is computed using a weighted Gaussian-style distance from the estimated background model.

The data cost is

```text
E_bg = wy  (Y - mu_Y)^2  (2  var_Y)
     +      (Cb - mu_Cb)^2  (2  var_Cb)
     +      (Cr - mu_Cr)^2  (2  var_Cr)
```

where

 `wy` is the luma weight.
 `mu` is the background mean.
 `var` is the background variance.
 A small epsilon is used for numerical stability.

A high background energy means the pixel is poorly explained by the green-screen model and is more likely to be foreground.

### 4. Initial Matte

The initial matte is generated using a threshold on the background energy

```text
alpha = 1 if E_bg  Et
alpha = 0 otherwise
```

The threshold `Et` controls how aggressively pixels are classified as foreground. A lower threshold creates a more aggressive foreground cut, while a higher threshold keeps more pixels as background.

The final tuned threshold used in the project was

```text
Et = 30
```

### 5. MRF Prior and Iterative Refinement

A pure data-cost matte can be noisy because sensor noise, compression artefacts, colour spill and local lighting variation can create isolated misclassified pixels.

To reduce this, an MRF prior is added. The prior encourages neighbouring pixels to share the same label.

For each pixel, the 8-neighbourhood is examined. The MRF update compares the total energy of assigning the current pixel to background versus foreground

```text
E0 = E_bg + lambda  number_of_foreground_neighbours

E1 = E_fg + lambda  number_of_background_neighbours
```

The pixel is assigned to the label with lower energy.

The final tuned parameters were

```text
Et = 30
lambda = 100
```

The MRFStep BlinkScript node was repeated 8 times to progressively refine the matte.

## BlinkScript Kernels

The project uses several custom BlinkScript kernels

 Kernel          Purpose                                                
 --------------  ------------------------------------------------------ 
 `ReduceMean2x`  Repeated 2× reduction to compute globalcrop means     
 `SquareOne`     Squares per-channel residuals for variance computation 
 `DataCostYUV`   Computes the per-pixel Bayesian background energy      
 `InitMatte`     Creates the initial binary matte from the data cost    
 `MRFStep`       Performs one ICMMRF refinement iteration              
 `SqError`       Computes per-pixel squared error for MSE               
 `SSIM_Moments`  Computes image moments needed for SSIM                 
 `SSIM_XY`       Computes cross moment for SSIM                         
 `SSIM_Final`    Computes final global SSIM value                       

## Evaluation

The Bayesian keyer was compared against

1. Keylight
2. U-Net output
3. Ground-truth matte

The evaluation used

 Mean Squared Error
 Structural Similarity Index

### Quantitative Result Summary

 Method              Average MSE  Average SSIM  Ranking 
 ------------------  ----------  -----------  ------- 
 Keylight                0.00953       0.96394  1       
 U-Net                   0.01876       0.92087  2       
 Bayesian MAP Keyer      0.01916       0.91888  3       

Keylight achieved the best overall result, which is expected because it is a production-grade keying tool. The U-Net ranked second, while the Bayesian MAP keyer achieved results close to the U-Net output.

Frame 49 showed degradation across all methods. This was likely caused by motion blur near hair and arm edges, fine hair strands against the green screen, and green spill reflected by the metallic dress.

## Results and Observations

The Bayesian keyer successfully produced a usable binary foreground matte using only a statistical background model and an MRF smoothness prior.

The strongest parts of the implementation are

 Clear probabilistic formulation.
 Robust use of YCbCr chroma information.
 Fully implemented BlinkScript processing pipeline.
 MRF-based spatial refinement.
 Quantitative comparison against U-Net and Keylight.

The main weakness is that the output matte is binary. Semi-transparent regions such as hair, motion blur and soft edges cannot be represented accurately with only `alpha = 0` or `alpha = 1`.

## Limitations

1. Binary Matte Only

   The method produces a hard binary matte. It cannot accurately represent semi-transparent hair, motion blur or soft edges.

2. Static Background Model

   The background mean and variance are estimated from one crop. This does not model spatial variation, lighting gradients or screen flicker.

3. Diagonal Covariance

   The background model treats Y, Cb and Cr channels independently. Cross-channel correlations are ignored.

4. No Temporal Coherence

   Each frame is processed independently, so temporal flickering can occur across a sequence.

5. Manual Parameter Tuning

   The threshold `Et` and smoothness weight `lambda` were tuned manually through visual inspection.

## Possible Improvements

Future improvements could include

 Soft alpha estimation with continuous alpha values in `[0,1]`.
 Spatially varying background models.
 Full covariance Gaussian modelling.
 Temporal MRF prior using previous-frame mattes.
 Automatic tuning of `Et` and `lambda`.
 Better modelling of green spill around reflective clothing and hair.
 Comparison with additional matting algorithms such as closed-form matting or Bayesian matting.

## Tools Used

 Nuke
 BlinkScript
 Python  PyTorch model comparison
 U-Net matte inference
 Keylight
 YCbCr colour-space modelling
 MSE and SSIM evaluation

## Skills Demonstrated

 Probabilistic modelling
 Bayesian MAP estimation
 Image compositing theory
 Alpha matte estimation
 Nuke node-graph development
 BlinkScript kernel programming
 Markov Random Field modelling
 Iterated Conditional Modes optimisation
 Colour-space analysis
 Quantitative image-quality evaluation
 MSE and SSIM implementation
 Comparison against deep-learning and production-grade keying methods

## Repository Notes

Large video files, generated frame sequences and heavy intermediate outputs are not included in this repository. The repository focuses on the implementation, methodology, scripts, selected results and project presentation.
