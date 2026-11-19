# Depth Consistency for UAV Video

Research project on improving the temporal consistency of dense depth maps
estimated from UAV video while accounting for camera motion.

Frame-by-frame monocular depth estimation often produces flickering: the
predicted depth of a static scene changes between consecutive frames. This
project investigates a geometry-aware optimization method that combines
single-frame depth regression, visual-inertial motion estimation, semantic
information, and variational regularization.

> **Project status:** research and implementation in progress. The repository
> currently contains the project specification; experimental results and
> implementation details will be added as the pipeline is developed.

## Research objective

The objective is to reduce temporal inconsistency while preserving the
absolute accuracy of the depth estimate. The method is designed to:

- use short-term camera motion estimated by visual-inertial odometry (VIO);
- reproject the current depth map into the previous camera frame;
- reduce the influence of unreliable regions such as occlusions and dynamic
  objects;
- preserve depth discontinuities at semantic object boundaries; and
- refine the result with a differentiable variational optimization step.

## Method overview

For a pixel \(x = (u, v)^T\), camera intrinsics \(K\), and current depth
\(D_t(x)\), the corresponding 3D point is

```text
P_t(x) = D_t(x) K⁻¹ x~
```

The relative camera motion from frame \(t\) to frame \(t-1\) is represented by
an element of \(SE(3)\):

```text
T_(t-1←t) = [ R  t ]
             [ 0  1 ]
```

The point is then warped into the previous image using

```text
x' = π(K (R D_t(x) K⁻¹ x~ + t))
```

where \(π\) is perspective division. This geometric warp supplies the
inter-frame connection needed by the temporal consistency term.

### Processing pipeline

1. **Depth regression** — estimate an initial depth map
   \(D_t^{raw} = f_\theta(I_t)\).
2. **Semantic and dynamic-scene analysis** — estimate semantic class
   probabilities and the probability that each pixel belongs to a dynamic
   object.
3. **Region categorization** — classify pixels as static background,
   occlusion, or dynamic regions using photometric residuals, motion
   uncertainty, and dynamic-object probabilities.
4. **Geometric reprojection** — warp the previous refined depth map into the
   current frame using the VIO motion estimate.
5. **Variational refinement** — optimize the current depth map using data,
   temporal, and semantic-aware spatial terms.

## Energy functional

The refined depth map is obtained by minimizing

```text
E(D_t) = ∫Ω [
    λ₁ Ψε(D_t - D_t^raw)
  + λ₂ M_temp Ψε(D_t - D̂_(t-1)(x'))
  + λ₃ W_sem Ψε(∇D_t)
] dx
```

Here:

- \(Ψ_\epsilon(s) = \sqrt{s^2 + \epsilon^2}\) is the Charbonnier robust loss;
- \(M_{temp}(x)\) is a temporal-confidence weight that suppresses unreliable
  temporal transfer in occluded and dynamic regions;
- \(W_{sem}(x)\) is a spatially adaptive weight that helps preserve semantic
  boundaries; and
- \(λ_1, λ_2, λ_3\) control the data, temporal, and spatial terms.

The optimization is implemented as an iterative gradient-descent scheme. The
analytical Jacobian of the reprojection with respect to depth can be used
explicitly; an approximation \(J_{warp} \approx 1\) will be evaluated as part
of the ablation study.

## Evaluation

### Datasets

The initial evaluation protocol targets:

- [KITTI](https://www.cvlibs.net/datasets/kitti/)
- [EuRoC MAV](https://projects.asl.ethz.ch/datasets/doku.php?id=kmavvisualinertialdatasets)

These datasets provide video sequences with geometric reference data suitable
for evaluating depth accuracy and temporal behavior.

### Metrics

Absolute depth accuracy will be measured with:

- AbsRel
- SqRel
- RMSE
- \(\delta < 1.25\)
- \(\delta < 1.25^2\)
- \(\delta < 1.25^3\)

Temporal behavior will be measured with a consistency error over static
regions:

```text
TC(D_t, D_(t-1)) =
    1 / |Ω_stat| · Σ_{x∈Ω_stat}
    |D_t(x) - D̂_(t-1)(W(x; D_t, T_(t-1←t)))|
```

Temporal flickering will also be evaluated as the depth variance of static
pixels along their trajectories.

### Planned ablation study

The following configurations will be compared:

1. Full method.
2. Without the temporal-confidence map \(M_{temp}\).
3. Without the semantic spatial weight \(W_{sem}\).
4. Without VIO-based reprojection.
5. Full reprojection Jacobian versus \(J_{warp} \approx 1\).

The target is to reduce temporal consistency error relative to frame-by-frame
depth estimation while keeping any degradation in absolute depth accuracy
within the tolerance established by pilot experiments.

## Robustness and fallback behavior

VIO can degrade on dynamic or visually challenging scenes. Motion uncertainty
is therefore included in the region categorization. When the uncertainty
exceeds the selected threshold, the temporal term is disabled for the affected
pixels and the method falls back to frame-wise depth estimation with spatial
regularization.

## Roadmap

- [ ] Review related work and finalize the comparison protocol.
- [ ] Implement the depth regression, semantic analysis, and region
  categorization stages.
- [ ] Implement VIO-based reprojection and temporal weighting.
- [ ] Implement the variational depth refinement.
- [ ] Run experiments and the ablation study on KITTI and EuRoC.
- [ ] Document results and prepare the final research report.

## References

- Godard et al. — [Digging Into Self-Supervised Monocular Depth Estimation](https://openaccess.thecvf.com/content_ICCV_2019/html/Godard_Digging_Into_Self-Supervised_Monocular_Depth_Estimation_ICCV_2019_paper.html), ICCV 2019.
- Luo et al. — [Consistent Video Depth Estimation](https://dl.acm.org/doi/10.1145/3450626.3459887), SIGGRAPH 2021.
- Casser et al. — [Depth Prediction Without the Sensors](https://ojs.aaai.org/index.php/AAAI/article/view/4918), AAAI 2019.
- Qin et al. — [VINS-Mono](https://github.com/HKUST-Aerial-Robotics/VINS-Mono), an optimization-based monocular visual-inertial state estimator.
- Campos et al. — [ORB-SLAM3](https://github.com/UZ-SLAMLab/ORB_SLAM3), a visual, visual-inertial, and multi-map SLAM system.
