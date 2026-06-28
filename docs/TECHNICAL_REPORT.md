# Technical Report

## Subject-Specific Sagittal Human Motion Reconstruction and Validation in MATLAB/Simscape

**Author:** Renan Arda Carlak

---

## Abstract

This project implements an end-to-end workflow for reconstructing measured human postural motion in MATLAB and Simscape Multibody. A full-body motion-capture recording containing 23 anatomical segments is converted into joint-center trajectories, a subject-specific N-pose reference, sagittal segment inclinations, and relative joint rotations. The calculated motion drives a multibody human model whose segment lengths are parameterized from the measured geometry.

A dedicated validation layer compares reconstructed sagittal joint-center coordinates with the original motion-capture coordinates, cross-checks position-derived angles against sagittal angles derived from segment-orientation quaternions, and decomposes parent-child link-vector errors into fixed geometric bias and time-varying residual components. The results show close agreement in joint-angle trajectories and sub-millimeter reconstruction discrepancies along the spinal chain. The largest upper-limb position differences are primarily caused by fixed anatomical frame and link-placement offsets rather than dynamic angle errors.

## 1. Objective and application context

The aim is to reconstruct postural motion recorded during an Achilles-tendon vibration experiment. The selected interval covers 45-65 s of the measurement, including the first vibration period between 50 and 60 s. The reconstruction provides two complementary visualizations:

- a subject-scaled Simscape Multibody representation driven by calculated sagittal joint rotations,
- a direct stick-figure animation based on measured joint-center coordinates.

The project focuses on the engineering pipeline from measurement data to a transparent and customizable multibody representation.

## 2. Input data and coordinate system

The input is a MATLAB structure converted from a motion-capture recording.

- Sampling frequency: `100 Hz`
- Number of body segments: `23`
- Analysis interval: frames `4500:6500`
- Global coordinate convention:
  - `+x`: forward,
  - `+y`: leftward,
  - `+z`: upward.

For each frame, the motion-capture structure provides:

- global proximal joint-center positions,
- segment-orientation quaternions,
- segment and joint labels.

The dataset used in this project was anonymized by removing the original participant filename, recording path, and recording date.

## 3. Processing pipeline

### 3.1 Joint-center extraction

The main MATLAB workflow reads the ordered motion-capture segment labels and assigns each triplet of global position values to the corresponding body segment. This produces a structured set of trajectories for the pelvis, spinal levels, upper limbs, lower limbs, feet, and toe roots.

### 3.2 N-pose reference

The Simscape model is constructed in an N-pose. The source recording contains a T-pose reference, so the arm coordinates in the reference frame are rearranged to form a geometrically consistent N-pose while preserving measured arm-segment lengths.

### 3.3 Sagittal segment inclinations

For a segment vector projected onto the sagittal plane:

```math
\mathbf{v}_{xz}(t)
=
\begin{bmatrix}
v_x(t) \\
v_z(t)
\end{bmatrix}
```

the global segment inclination is calculated as:

```math
\phi(t)
=
\mathrm{atan2}\left(v_z(t),\,v_x(t)\right)
```

The rotation relative to the N-pose is:

```math
\theta(t)
=
-\left[
\phi(t)-\phi_N
\right]
```

The negative sign follows the rotation convention used by the Simscape model.

### 3.4 Relative joint rotations

The model is represented as a serial multibody chain. Relative joint rotations are therefore calculated by subtracting the orientation of the proximal segment from that of the distal segment.

Examples include:

```math
\theta_{\mathrm{knee}}
=
\theta_{\mathrm{lower\,leg}}
-
\theta_{\mathrm{upper\,leg}}
```

```math
\theta_{\mathrm{elbow}}
=
\theta_{\mathrm{forearm}}
-
\theta_{\mathrm{upper\,arm}}
```

```math
\theta_{\mathrm{L3}}
=
\theta_{\mathrm{L3\,segment}}
-
\theta_{\mathrm{L5\,segment}}
```

These relative rotations are applied to the corresponding revolute joints in the Simscape Multibody model.

## 4. Multibody model

The Simscape Multibody model represents the major segments of the spine, arms, pelvis, legs, feet, and toe roots. Segment geometry is parameterized from the motion-capture reference configuration. The pelvis translation and sagittal joint rotations are prescribed from the measurement-derived signals.

The model is intended as a kinematic motion-reconstruction environment. It is not used here to estimate validated joint torques, muscle forces, or predictive balance-control responses.

![Simscape Multibody model overview](<figures/simscape_model_overview.png>)

*Figure 1. Overview of the subject-scaled Simscape Multibody model architecture.*


## 5. Visualization

The measured motion is visualized in two forms:

1. A Simscape Multibody animation showing the subject-scaled mechanical representation.
2. A three-dimensional stick figure assembled directly from motion-capture joint-center coordinates and viewed in the sagittal plane.

The two views provide a direct comparison between measured geometry and the multibody reconstruction.

## 6. Validation methodology

### 6.1 Joint-center reconstruction

The Simscape model is solved kinematically for every frame. Model joint-center coordinates are recovered in the sagittal plane and compared with the measured motion-capture coordinates.

For joint center $i$:

```math
\mathbf{e}_i(t)
=
\mathbf{p}_{i,\mathrm{model}}(t)
-
\mathbf{p}_{i,\mathrm{MoCap}}(t)
```

```math
e_i(t)
=
\sqrt{
e_{i,x}^{2}(t)
+
e_{i,z}^{2}(t)
}
```

The reported metrics are mean error, maximum error, and RMSE:

```math
\mathrm{RMSE}_i
=
\sqrt{
\frac{1}{N}
\sum_{t=1}^{N}
e_i^{2}(t)
}
```

![Joint-center error summary](<figures/Validation - Position summary.png>)

*Figure 2. Sagittal-plane joint-center error summary, including maximum, mean, and RMSE values.*

![Joint-center position errors over time](<figures/Validation - Position errors.png>)

*Figure 3. Sagittal-plane joint-center position error over time for each anatomical point.*

### 6.2 Quaternion-based angle cross-validation

A second sagittal angle estimate is derived from motion-capture segment-orientation quaternions. The segment-local anatomical axis is identified from the original T-pose position vector and T-pose quaternion. The axis is then rotated by the quaternion of each measured frame and referenced to the N-pose direction.

The position-derived and quaternion-derived angle trajectories are compared through:

- RMSE,
- mean absolute error,
- maximum absolute error,
- correlation,
- time-series overlays.

![Sagittal angle comparison](<figures/Validation - Angle comparison.png>)

*Figure 4. Comparison of position-derived model angles and quaternion-derived sagittal angles.*

![Sagittal angle validation summary](<figures/Validation - Angle summary.png>)

*Figure 5. Summary of angular validation metrics: maximum absolute error, mean absolute error, and RMSE.*

![Sagittal angle errors over time](<figures/Validation - Angle errors.png>)

*Figure 6. Sagittal-angle error over time for the evaluated joints.*

### 6.3 Parent-child link-vector decomposition

Absolute joint-center errors can accumulate across a serial chain. To localize their source, each parent-child vector is compared directly:

```math
\mathbf{r}_{\mathrm{model}}(t)
=
\mathbf{p}_{\mathrm{child,model}}(t)
-
\mathbf{p}_{\mathrm{parent,model}}(t)
```

```math
\mathbf{r}_{\mathrm{MoCap}}(t)
=
\mathbf{p}_{\mathrm{child,MoCap}}(t)
-
\mathbf{p}_{\mathrm{parent,MoCap}}(t)
```

```math
\Delta \mathbf{r}(t)
=
\mathbf{r}_{\mathrm{model}}(t)
-
\mathbf{r}_{\mathrm{MoCap}}(t)
```

The mean link-vector difference is treated as a fixed geometric bias:

```math
\overline{\Delta \mathbf{r}}
=
\frac{1}{N}
\sum_{t=1}^{N}
\Delta \mathbf{r}(t)
```

The dynamic residual is:

```math
\Delta \mathbf{r}_{\mathrm{dynamic}}(t)
=
\Delta \mathbf{r}(t)
-
\overline{\Delta \mathbf{r}}
```

This decomposition distinguishes static landmark and frame-placement differences from motion-dependent kinematic errors.

![Link error decomposition](<figures/Validation - Link error decomposition.png>)

*Figure 7. Parent-child link-vector error decomposition into total RMSE, fixed geometric bias, and dynamic residual RMSE.*

![Link-vector errors over time](<figures/Validation - Link error time series.png>)

*Figure 8. Parent-child link-vector error over time before and after removal of the fixed geometric bias.*

![Mean sagittal link lengths](<figures/Validation - Link length comparison.png>)

*Figure 9. Mean sagittal parent-child link lengths in the motion-capture data and the Simscape model.*

## 7. Results and interpretation

### 7.1 Joint-angle validation

The position-derived and quaternion-derived sagittal angle trajectories overlap closely across the spine, hips, knees, shoulders, and elbows. For most non-ankle joints, the differences are at numerical-noise scale. The ankle comparisons show the largest angular differences, but remain below approximately `0.1 deg`.

This agreement supports the internal consistency of the sagittal-angle extraction and relative-joint formulation with the motion-capture segment-orientation data.

### 7.2 Spinal chain reconstruction

The pelvis-to-head chain shows very small joint-center and link-vector discrepancies. The pelvis is a prescribed translational reference, while the L5, L3, T12, T8, neck, and head trajectories are reconstructed with sub-millimeter numerical error.

This indicates consistent segment geometry, joint topology, and relative-angle transfer through the spinal chain.

### 7.3 T8-to-shoulder-root geometry

The T8-to-left-shoulder-root and T8-to-right-shoulder-root links remain below approximately one millimeter of error. This indicates that the upper-thorax-to-scapula-root mapping is consistent after matching the model and motion-capture anatomical points.

### 7.4 Shoulder-to-upper-arm offsets

The dominant upper-limb discrepancy begins at the shoulder-root-to-upper-arm connection:

- right side: approximately `29 mm`,
- left side: approximately `27 mm`.

The link decomposition shows that nearly all of this difference is fixed geometric bias, while the dynamic residual is small. The Simscape shoulder-root and shoulder-joint frames therefore differ from the corresponding motion-capture landmark geometry by an almost constant sagittal offset.

### 7.5 Upper-arm-to-forearm offsets

The upper-arm-to-forearm connections show an additional error of approximately `17 mm` on both sides. The mean sagittal link lengths remain similar, indicating that the discrepancy is not primarily caused by segment-length mismatch.

The dominant fixed bias is consistent with a small constant difference in the link's zero orientation or local frame alignment. For an upper-arm length of roughly `320 mm`, a `17 mm` endpoint difference corresponds to an angular offset on the order of three degrees.

### 7.6 Forearm-to-hand connections

The right forearm-to-hand link is reconstructed closely, with a sub-millimeter error. The left side contains a smaller fixed difference of approximately `2 mm`. These results indicate that wrist placement contributes little to the right-hand error and only a limited amount to the left-hand error.

### 7.7 Accumulation of the right-hand position error

The absolute right-hand difference approaches approximately `40 mm`. This is not produced by one isolated 40 mm error. It results from vector accumulation of:

1. the shoulder-root-to-upper-arm fixed offset,
2. the upper-arm-to-forearm fixed orientation/frame offset,
3. the much smaller forearm-to-hand difference.

Because these components have different directions, their magnitudes do not add arithmetically. Their vector combination produces the observed distal hand error.

### 7.8 Lower-limb chain

The main lower-limb difference appears in the pelvis-to-hip-joint connection and is approximately `4-5 mm` on both sides. The upper-leg-to-lower-leg and lower-leg-to-foot connections remain below approximately `0.5 mm`.

The lower-limb angles and long-segment geometry are therefore reconstructed consistently. The dominant difference is a small fixed pelvis-to-hip landmark offset rather than a time-varying joint-angle error.

The left foot-to-toe connection contains an additional fixed difference of approximately `7 mm`, explaining why the left-toe absolute error is higher than the remaining lower-limb points.

### 7.9 Fixed bias components

The fixed-bias components show the direction of each geometric difference.

- The shoulder-to-upper-arm links contain a dominant vertical component and, on the right side, an additional anterior-posterior component.
- The upper-arm-to-forearm links contain a predominantly anterior-posterior bias.
- The pelvis-to-hip offsets are small and largely constant.
- The left foot-to-toe link contains a distinct fixed length and direction difference.

![Signed fixed bias components](<figures/Validation - Link bias components.png>)

*Figure 10. Signed fixed geometric bias components for each sagittal parent-child link.*

### 7.10 Dynamic residuals

For the links with the largest absolute errors, the dynamic residual is much smaller than the fixed bias:

- shoulder-to-upper-arm: roughly `1 mm` dynamic residual versus `27-30 mm` fixed bias,
- upper-arm-to-forearm: roughly `3 mm` dynamic residual versus approximately `17 mm` fixed bias,
- pelvis-to-hip: sub-millimeter dynamic residual versus approximately `4-5 mm` fixed bias,
- left foot-to-toe: sub-millimeter dynamic residual versus approximately `7 mm` fixed bias.

This indicates that the model generally follows the temporal motion pattern correctly. Most remaining joint-center differences arise from fixed geometry and landmark definitions.

## 8. Technical diagnosis

The validation supports the following diagnosis:

### Components verified as consistent

- sagittal angle extraction from position vectors,
- quaternion-based angle cross-validation,
- relative-joint-angle calculation,
- spinal-chain topology,
- hip-knee-ankle kinematic transfer,
- upper-thorax-to-shoulder-root mapping,
- temporal tracking of measured postural motion.

### Remaining fixed geometric differences

- shoulder root to shoulder joint,
- upper-arm local zero orientation or frame alignment,
- pelvis center to hip-joint placement,
- left foot to toe-root placement,
- a smaller left forearm-to-hand difference.

## 9. Scope of the conclusions

The project demonstrates a transparent and quantitatively evaluated kinematic reconstruction workflow. It does not claim:

- independent experimental validation against optical motion capture,
- predictive simulation of postural responses,
- validated inverse dynamics,
- muscle-force estimation,
- full three-dimensional inverse kinematics.

The angle comparison is an internal cross-validation between motion-capture position and orientation outputs. The joint-center analysis evaluates how the measured kinematics are represented by the sagittal multibody geometry.

## 10. Conclusion

The model accurately reconstructs sagittal joint-angle trajectories and reproduces the spinal and lower-limb serial chains with small dynamic residuals. The dominant upper-limb and selected distal-limb position differences are attributable to fixed anatomical frame and link-placement offsets rather than time-varying kinematic errors.

The resulting project is a complete computational biomechanics case study that combines motion-capture processing, subject-specific parameterization, multibody modeling, visualization, quantitative validation, and structured error diagnosis.


---

**Copyright © 2021–2026 Renan Arda Carlak. All rights reserved.**

This report and its associated figures are covered by the repository’s [Proprietary Evaluation-Only License](../LICENSE.md).
