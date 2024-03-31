---
layout: post
title:  "Problem Statement"
date:   2024-03-31 14:10:00 +0030
---


Human surface anatomy implies a deformable continuous body with occlusions. Many algorithms independently tackle subsets of this, but I did not find an out-of-the-box fit for long-term feature detection & tracking. The closest related deep learning models for this problem are the pose detection and the body mesh AR models.


I explored full-body pose estimation, ViT-Pose, DWPose, RTMPose, 3D Hand mesh, and found that the granularity needed for our purpose cannot be interpolated from the fixed number of keypoints (~17) tracked in the popular and openly available datasets. One of the transformer-based models for matching geospatial instances over time tackles a similar problem, and I might explore it in the future. 

We want to make the scanning process feel natural and convenient while maximizing data collection. Scale-invariant image registration is, therefore, a critical problem to solve here. Registration is a process commonly used in medical image analysis tools like ITK for alignment. In short, the app should be able to match and align a close-up picture of a mole on the wrist to images of the entire arm.


In addition to that, the other problems for the VisionBackend to solve are variability in appearance and pose, occlusions, motion, and blur.


In the current form, I am using SIFT descriptors along with some heuristics like a continuous body model to solve these. I will be exploring various permutations and writing about the same in the next posts.

