# RSA-LIVO

**Robust Semantic-Aided LiDAR-Inertial-Visual Odometry for Unstructured and Sensor-Degraded Environments**

Yingying Lei, Naohiko Hanajima, Yoshinori Fujihira, and Jun Dai
System Control Laboratory, Muroran Institute of Technology, Japan

> The manuscript is currently under review at *IEEE Transactions on Instrumentation and Measurement*.

## Release status

| Item | Status |
|---|---|
| Project documentation (this page): method overview, datasets and sequences, nominal parameters, evaluation protocol | Available |
| Source code of RSA-LIVO (estimator, Module A, Module B, semantic front end) | Released upon acceptance |
| Configuration files of the reported experiments | Released upon acceptance |
| Evaluation and statistics scripts | Released upon acceptance |

The released version will be tagged to correspond to the accepted manuscript.

## Overview

![RSA-LIVO system architecture](docs/framework.png)

RSA-LIVO is a semantic-aided LiDAR-inertial-visual odometry framework built on the error-state iterated Kalman filter (ESIKF) of FAST-LIVO2. It represents scene-dependent observation reliability explicitly in the measurement covariance of the LiDAR point-to-plane observations.

- **Semantic perception front end.** An HRNet-W48+OCR network runs asynchronously in a separate GPU process. It provides per-pixel labels, confidence scores, and normalized prediction entropy, which are associated with the LiDAR observations by timestamp and image projection.
- **Module A: Adaptive Covariance Inflation (ACI).** Camera-side evidence, derived from fixed class priors and prediction entropy, is fused with LiDAR-side residual-consistency evidence by Dempster-Shafer combination. The fused reliability q_j in [0, 1] scales the nominal covariance of each LiDAR observation: R_eff,j = R_phys,j [1 + λv (1 − q_j)].
- **Module B: Geometric-Consistency Gating.** Semantic-boundary LiDAR observations are tested with a χ² criterion (one degree of freedom, 0.95 quantile) on the pre-inflation residual variance. Observations that fail the test are excluded from the current update.
- Visual photometric residuals keep their constant covariance. The voxel map stores geometric planes only.

## Software environment and dependencies

- **Estimator:** RSA-LIVO is built on [FAST-LIVO2](https://github.com/hku-mars/FAST-LIVO2) and uses its ROS-based dependencies (Eigen, PCL, OpenCV, Sophus, and the Livox driver for the MID-360 data).
- **Semantic front end:** PyTorch with CUDA, using [HRNet-Semantic-Segmentation](https://github.com/HRNet/HRNet-Semantic-Segmentation) (HRNet-W48+OCR) initialized from ImageNet-pretrained HRNetV2-W48 weights.
- **Evaluation:** [evo](https://github.com/MichaelGrupp/evo) for trajectory evaluation.
- **Experiment hardware:** Intel Core Ultra 9 285K, 64 GB RAM, NVIDIA RTX A5000, Ubuntu 22.04. Each external baseline was run in its released software environment in a separate container.

Exact package versions and build instructions will be provided with the code release.

## Datasets

The datasets are not redistributed in this repository. Please obtain them from their providers:

- **M3DGR:** https://github.com/sjtuyinjie/M3DGR
- **RELLIS-3D:** https://github.com/unmannedlab/RELLIS-3D

| Dataset | Sequences | Sensors used | Reference trajectory | Output point |
|---|---|---|---|---|
| M3DGR | Dynamic03, Dynamic04, Grass01, Grass02, Z-Rough, Occlusion03, Occlusion04 | Livox MID-360 (10 Hz), MID-360 internal IMU (200 Hz), RealSense D435i RGB (640×480, 30 Hz) | RTK-GNSS positions, 15 Hz | MID-360 internal IMU |
| RELLIS-3D | Rellis0001–Rellis0004 | Ouster OS1-64 (10 Hz), VectorNav VN-300 IMU, Basler acA1920-50gc RGB (1920×1200, 10 Hz) | VN-300 GNSS/INS solution without RTK | VN-300 IMU frame |

- All extrinsics are taken from the official calibration of each dataset. On RELLIS-3D, per-sequence camera extrinsics are used.
- The segmentation network was trained on the RELLIS-3D training split (19 classes) and applied to M3DGR without fine-tuning.
- The evaluation intervals and calibration files used for each sequence will be included in the release.

## Nominal parameters

All parameters were fixed before the experiments and are identical for all sequences.

| Parameter | Symbol | Value |
|---|---|---|
| Covariance-inflation scale | λv | 1.5 |
| Entropy-decay parameter | β | 3 |
| Class-prior strength | γ | 1 |
| Fixed ignorance mass | u | 0.1 |
| Conflict threshold | K_th | 0.95 |
| Geometric gate | χ²₁(0.95) | 3.84 |
| Semantic association tolerance | – | 40 ms |
| Semantic frame stride | – | every second camera frame |
| Image size for inference | – | 960×608, FP16 |

### Class-conditioned reliability priors

| ID | Class | α_prior | ID | Class | α_prior |
|---|---|---|---|---|---|
| 0 | void / dirt | 0.5 | 10 | log | 0.7 |
| 1 | grass | 0.6 | 11 | person | 0.2 |
| 2 | tree | 0.7 | 12 | fence | 1.0 |
| 3 | pole | 0.9 | 13 | bush | 0.6 |
| 4 | water | 0.2 | 14 | concrete | 1.0 |
| 5 | sky | 0.1 | 15 | barrier | 0.9 |
| 6 | vehicle | 0.2 | 16 | puddle | 0.3 |
| 7 | object | 0.4 | 17 | mud | 0.5 |
| 8 | asphalt | 1.0 | 18 | rubble | 0.5 |
| 9 | building | 1.0 | | | |

## Evaluation protocol

- **Pose association:** estimated and reference poses are associated by nearest timestamp with a tolerance of 40 ms and zero time offset.
- **Alignment:** a single SE(3) transformation without scale.
- **APE:** translational absolute pose error, RMSE per run.
- **RTE:** position-only relative trajectory error over a 1 s window, RMSE per run. Reference positions are linearly interpolated between samples at most 0.25 s apart.
- **Run validity:** determined only by predeclared execution-integrity criteria (configuration identity, accepted input domain, trajectory completeness, estimator-update consistency, and output integrity). Valid runs are retained regardless of their error.
- **Statistics:** one APE and one RTE value per run. FAST-LIVO2 and RSA-LIVO are compared per sequence with two-sided Welch tests, and the Holm procedure is applied separately to the APE and RTE comparisons.

## Experiments in the paper

| Experiment | Configurations | Runs |
|---|---|---|
| Primary comparison | RSA-LIVO, FAST-LIVO2, SR-LIVO, R3LIVE, Ground-Fusion2, VINS-Fusion, LIO-SAM | 5 runs per method and sequence |
| Controlled ablation | OFF, HARD_REJECTION_CONTROL, GENERIC_WEIGHTING, A_ONLY, B_ONLY, FULL_RSA_LIVO | 3 runs per configuration and sequence |
| Parameter sensitivity | λv ∈ {0.5, 1, 2, 3}, β ∈ {1, 2, 4, 5}, γ ∈ {0, 0.5} | 1 run per setting and sequence |

All experiments use the eleven sequences listed above. The results are reported in the paper.

## Citation

The citation will be added after publication.

## License

The license will be specified with the code release and will be compatible with the license of FAST-LIVO2.

## Contact

Please open an issue in this repository.
