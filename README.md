# Summary
Reproduction and extension of a LiDAR-based place recognition and localization method for autonomous vehicles. This README provides a concise overview of my Master's project, including the system pipeline, dataset adaptation, implementation improvements, experimental results, and ongoing work. It is intended to help readers quickly understand the scope, methodology, and progress of the project without going into all implementation details.

## System Pipeline
### Overall Pipeline

<p align="center">
  <img src="images/Overall%20Pipeline.drawio.png" width="900">
</p>
The pipeline converts LiDAR point clouds and vehicle poses into Bird's-Eye-View (BEV) representations for model training and localization. During training, a shared feature encoder learns BEV representations for place retrieval, while a refinement module estimates the relative pose between a query and its matched anchor. The trained model is then used to retrieve the most relevant anchor and estimate the vehicle pose during localization.

- `bev_manifest.csv` : Records each generated BEV frame together with its timestamp and global pose (x, y, yaw).
- `anchors.csv` : Stores the selected reference BEV frames (anchors) and their corresponding global poses.
- `pairs.csv` : Defines query–anchor training pairs, including place-recognition labels, relative pose offsets (dx, dy, dyaw), and a validity mask for pose refinement.
Shared BEV Feature Encoder — A lightweight CNN-based encoder shared by query and anchor BEVs. It converts each BEV image into a compact 256-dimensional feature descriptor used for similarity matching. The original architecture uses four convolutional blocks followed by global average pooling and descriptor normalization.
Retrieval — Compares the query descriptor with the anchor database using feature similarity and selects the most relevant anchor as the coarse location estimate.
Refinement — Uses the query BEV and retrieved anchor BEV to estimate the local relative pose correction (dx, dy, dyaw) and its uncertainty, improving the coarse retrieval result into a more precise pose estimate.
### Pose Generation Pipeline

<p align="center">
  <img src="images/Pose%20Generation%20Pipeline.png" width="900">
</p>
