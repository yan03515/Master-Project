# Master Project
## Summary
Reproduction and extension of a **LiDAR-based place recognition and localization** method for autonomous vehicles. This README provides a concise overview of my Master's project, including the system pipeline, dataset adaptation, implementation improvements, experimental results, and ongoing work. It is intended to help readers quickly understand the scope, methodology, and progress of the project without going into all implementation details.

## System Pipeline

### Overall Pipeline

<p align="center">
  <img src="images/Overall%20Pipeline.drawio.png" width="900">
</p>

The pipeline converts LiDAR point clouds and vehicle poses into Bird's-Eye-View (BEV) representations for model training and localization. During training, a shared feature encoder learns BEV representations for place retrieval, while a refinement module estimates the relative pose between a query and its matched anchor. The trained model is then used to retrieve the most relevant anchor and estimate the vehicle pose during localization.

- `bev_manifest.csv` : Records each generated BEV frame together with its timestamp and global pose `(x, y, yaw)`.
- `anchors.csv` : Stores the selected reference BEV frames (anchors) and their corresponding global poses.
- `pairs.csv` : Defines query–anchor training pairs, including place-recognition labels, relative pose offsets `(dx, dy, dyaw)`, and a validity mask for pose refinement.
- **Shared BEV Feature Encoder** : A lightweight CNN-based encoder that converts query and anchor BEVs into compact feature descriptors for similarity matching.
- **Retrieval** : Compares the query descriptor with the anchor database using feature similarity and selects the most relevant anchor as the coarse location estimate.
- **Refinement** : Uses the query BEV and retrieved anchor BEV to estimate the local relative pose correction `(dx, dy, dyaw)` and its uncertainty, improving the coarse retrieval result into a more precise pose estimate.

### Pose Generation Pipeline

<p align="center">
  <img src="images/Pose%20Generation%20Pipeline.png" width="900">
</p>

The raw ground-truth poses are transformed from ECEF coordinates into a shared ENU coordinate frame and temporally aligned with the LiDAR timestamps. Position is interpolated linearly, while orientation is interpolated using quaternion SLERP before extracting the final yaw angle. The resulting `poses.csv` contains `(timestamp, x, y, yaw)` for each LiDAR frame.


`poses.csv` is required to associate each LiDAR frame with its global position and orientation. The LiDAR point clouds and `poses.csv` are then processed to generate the BEV images, `bev_manifest.csv`, `anchors.csv`, and `pairs.csv` used in the subsequent training and localization pipeline.

## Dataset Adaptation and Implementation Changes

### Dataset

This project uses [M2DGR](https://github.com/SJTU-ViSYS/M2DGR) as the primary experimental dataset. M2DGR includes a variety of outdoor environments, such as campus roads and urban streets, and provides LiDAR data with corresponding ground-truth trajectories for localization training and evaluation. In addition, M2DGR uses a **Velodyne VLP-32C LiDAR**, the same LiDAR model used on our laboratory autonomous vehicle platform. Therefore, this dataset better represents the conditions expected for future deployment and testing on the **NTUST campus**.
| Split | Sequences |
| --- | --- |
| Train | `gate_01`, `rotation_02`, `street_01`, `street_02`, `street_03`, `street_05`, `street_06`, `street_08`, `street_09`, `street_10` |
| Validation | `walk_01` |
| Test | `rotation_01`, `street_04` |

*Note: The sequences were split by route rather than by individual frames to reduce temporal data leakage and evaluate generalization across different trajectories. Spatial overlap between sequences may still exist.*

### Implementation Changes

- **Reconstructed `poses.csv` for M2DGR**  
  Converted the M2DGR ground-truth positions from ECEF to a shared ENU coordinate system and aligned them with LiDAR timestamps. Position was interpolated linearly, while orientation was interpolated using quaternion SLERP to generate `(timestamp, x, y, yaw)` for each LiDAR frame.
- **Added mask handling for pose refinement**  
  Samples outside the valid refinement range are still retained for place-recognition training, while being excluded from the pose-refinement loss. This allows retrieval learning to use more data without introducing unreliable pose-regression targets.
- **Improved InfoNCE negative-sample handling**  
  Since all selected M2DGR sequences were transformed into the same global ENU coordinate system in this project, frames from different sequences may correspond to nearby or overlapping locations. Spatially close cross-sequence samples are therefore excluded from the negative set to reduce false negatives during retrieval training.

## Results and Observed Issues

### Test Results

| Metric | `rotation_01` | `street_04` |
| --- | ---: | ---: |
| R@1 @ 2 m | 80.38% | 66.38% |
| R@1 @ 5 m | 95.74% | 93.81% |
| R@1 @ 10 m | 99.78% | 98.04% |
| R@5 @ 2 m | 97.37% | 78.00% |
| R@5 @ 5 m | 99.60% | 99.73% |
| R@5 @ 10 m | 100% | 100% |
| Retrieval CEP50 | 0.773 m | 1.487 m |
| Refined CEP50 | 0.683 m | 0.198 m |
| Translation RMSE | 1.994 m | 3.493 m |
| Rotation RMSE | 12.97° | 2.08° |

### GT vs Predicted Trajectories

<table>
  <tr>
    <td align="center">
      <img src="images/rotation_01%20trajectory.png" height="400"><br>
    </td>
    <td align="center">
      <img src="images/street_04%20trajectory.png" height="400"><br>
    </td>
  </tr>
</table>

### GOOD / BAD Retrieval Examples

<table>
  <tr>
    <td align="center">
      <img src="images/rotation_01%20GOOD.png" height="400">
    </td>
    <td align="center">
      <img src="images/street_04%20GOOD.png" height="400">
    </td>
  </tr>
  <tr>
    <td align="center">
      <img src="images/rotation_01%20BAD.png" height="400">
    </td>
    <td align="center">
      <img src="images/street_04%20BAD.png" height="400">
    </td>
  </tr>
</table>

### Observed Issues

- **Rotation and orientation ambiguity**  
  In `rotation_01`, localization becomes less stable under large orientation changes. BEV scenes with similar geometric structures may produce high feature similarity even when their orientations differ, leading to incorrect anchor retrieval and degraded pose estimation.
- **Long-distance retrieval outliers**  
  In `street_04`, most frames achieve accurate localization, but a small number of queries are matched to geometrically similar anchors located far from the correct position. These outliers significantly increase the overall translation RMSE despite the low median localization error.

## Current Work and Future Directions

### Current Work

- **Rotation Robustness Improvement**  
  Improving retrieval robustness under large orientation changes to reduce orientation ambiguity and incorrect anchor matching.

- **Retrieval Reliability**  
  Investigating methods to reduce long-distance wrong-anchor retrievals and improve localization stability in geometrically similar environments.

- **Adaptive Anchor Density**  
  Exploring adaptive anchor spacing based on localization uncertainty, with denser anchors in difficult regions and fewer anchors in reliable regions to balance localization robustness and map size.

### Future Directions

- **Real-Time Autonomous Localization and Navigation**

  Extending the current offline localization pipeline to process streaming LiDAR data in real time, and further integrating IMU sensor fusion toward real-time autonomous navigation on **NVIDIA Jetson Orin NX** within the **NTUST campus environment**.

  The system will be deployed and validated on our laboratory autonomous vehicle platform for real-world testing. The vehicle's mechanical structure, PC-side control program, and FPGA-based motor control system are documented in the [Autonomous-Vehicle](https://github.com/yan03515/Autonomous-Vehicle/tree/main) repository.
  
## Acknowledgment and References

This project uses the [M2DGR](https://github.com/SJTU-ViSYS/M2DGR) dataset for training and evaluation.

- J. Yin et al., "M2DGR: A Multi-sensor and Multi-scenario SLAM Dataset for Ground Robots," *IEEE Robotics and Automation Letters*, 2021.  
  [M2DGR GitHub Repository](https://github.com/SJTU-ViSYS/M2DGR)

Additional references related to the underlying localization method will be added when they become publicly available.
