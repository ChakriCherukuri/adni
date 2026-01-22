Below is a **practical, “solid” feature menu** you can extract into a **tabular dataset** from (a) the **per-scan JSON metadata** and (b) the **.nii image itself**, specifically with a **survival analysis** goal (time-to-conversion to AD, with censoring).

Structure this as:

1. **Per-scan features** (one row per scan/visit)
2. **Per-subject longitudinal summary features** (one row per subject)
3. A **recommended minimal baseline set** to start with (small N = 405)

---

## 1) Per-scan features from JSON metadata (scanner/protocol covariates)

These are valuable because ADNI has multi-site / multi-scanner variation and you don’t want imaging differences masquerading as disease signal.

### A. Scanner identity and site (categorical)

* `Manufacturer` (e.g., Siemens/GE/Philips)
* `ManufacturerModelName` or scanner model
* `DeviceSerialNumber` (often too specific; may overfit—use with caution)
* `Site` / `Center` (if present)
* `StationName`
* `SoftwareVersions`
* `BodyPartExamined` (should be head/brain)

**Engineering tip:** One-hot encode manufacturer/model/site; consider grouping rare categories.

### B. Sequence/protocol descriptors (categorical)

* `Modality` (MR)
* `SeriesDescription`, `ProtocolName`
* `ScanningSequence` (e.g., GR, SE)
* `SequenceVariant`, `ScanOptions`
* `MRAcquisitionType` (2D vs 3D)
* `ImageType` (e.g., ORIGINAL/DERIVED)
* `PulseSequenceDetails` (if present)

### C. Core acquisition parameters (numeric)

(Names vary by dataset; pull whatever exists)

* `MagneticFieldStrength` (1.5T vs 3T)
* `RepetitionTime` (TR)
* `EchoTime` (TE)
* `InversionTime` (TI) (often for MPRAGE)
* `FlipAngle`
* `PixelBandwidth`
* `EchoTrainLength`
* `NumberOfAverages` / `NEX`
* `ParallelReductionFactorInPlane` (acceleration factor)
* `PhaseEncodingDirection`
* `SAR` (specific absorption rate) if present

### D. Timing (use carefully)

* `AcquisitionTime` / `AcquisitionDateTime`
* `StudyDate`, `SeriesDate`

**Important:** Don’t use absolute dates as predictive features (they can proxy cohort/scanner upgrades). Convert them into:

* `months_since_subject_baseline` (safe and useful)
* optionally `age_at_scan` (if you have DOB elsewhere)

---

## 2) Per-scan features from the NIfTI itself (header + image-derived)

### A. NIfTI header geometry (robust and easy)

From `.nii` header (`pixdim`, shape, affine):

* `dim_x, dim_y, dim_z` (matrix size)
* `voxel_size_x, voxel_size_y, voxel_size_z` (mm)
* `voxel_volume_mm3 = vx*vy*vz`
* `FOV_x = dim_x*vx`, `FOV_y`, `FOV_z`
* `orientation` (RAS/LPS-like; from affine)
* `qform_code`, `sform_code` (sometimes correlates with processing pipelines)
* `datatype` (int16/float32)
* `slice_thickness` (if encoded separately)

These help catch protocol differences and can be strong confounders to control for.

---

## 3) Image quality / artifact features (high value baseline)

These features often explain performance differences and help you avoid “model learned scanner noise.”

### A. MRIQC-style QC metrics (recommended)

If you can run **MRIQC** (or implement similar):

* **SNR** (overall / tissue-specific if segmentation available)
* **CNR** (e.g., GM vs WM)
* **EFC** (entropy focus criterion; blur/ghosting proxy)
* **FBER** (foreground-background energy ratio)
* **CJV** (coefficient of joint variation; noise/bias proxy)
* **INU** / bias field strength estimates
* **WM2MAX** (white-matter to max intensity ratio)
* **Artifact indices** (ghosting, spikes) if available

Even a small set like `SNR`, `CNR`, `EFC`, `CJV` can be useful.

### B. Simple QC if you don’t want MRIQC yet

Compute after skull stripping (or approximate with thresholding):

* `brain_volume_vox` (brain mask voxel count)
* `mean_intensity_brain`, `std_intensity_brain`
* `p01/p50/p99` brain intensity percentiles
* `background_mean`, `background_std`
* `brain_to_background_ratio`

---

## 4) Core morphometric / neurodegeneration features (most predictive, most interpretable)

This is usually the strongest “tabular baseline” for AD progression.

### A. Global brain volumes (per scan)

Requires tissue segmentation (GM/WM/CSF), e.g., FAST/ANTs/FreeSurfer/FastSurfer:

* `TIV` / `ICV` (total intracranial volume)
* `total_brain_volume` (GM+WM)
* `GM_volume`, `WM_volume`, `CSF_volume`
* `brain_parenchymal_fraction = (GM+WM)/TIV`
* `ventricle_volume_total` (or lateral + third)

### B. AD-relevant ROI volumes (per scan)

From atlas/segmentation (FreeSurfer/FastSurfer/ANTs-atlas):

* Left/right **hippocampus volume**
* Left/right **amygdala volume**
* Left/right **entorhinal cortex volume/thickness**
* **parahippocampal gyrus** volume/thickness
* **inferior temporal**, **middle temporal** volume/thickness
* **fusiform** volume/thickness
* **temporal pole**
* **posterior cingulate**, **precuneus**
* **inferior parietal**
* **lateral ventricles** (L/R), **3rd ventricle**

### C. Cortical thickness summary features (per scan)

If you have surface-based measures:

* `mean_cortical_thickness`
* `AD_signature_thickness_mean` (average thickness across an “AD signature” region set)
* regional thickness values for the ROIs above

### D. Normalized and asymmetry features (cheap + strong)

Normalization:

* `hippocampus_L_norm = hippo_L / TIV`
* `hippocampus_R_norm = hippo_R / TIV`
* `ventricle_norm = ventricle / TIV`

Asymmetry (often informative):

* `hippocampus_asym = (L - R) / (L + R + eps)`
* similar for amygdala, temporal lobe ROIs

---

## 5) Radiomics / texture features (useful, but control dimensionality)

These can add predictive signal beyond volumes, but they can explode into thousands of columns.

### A. First-order intensity stats (within ROIs)

Compute within hippocampus / temporal GM / whole GM:

* mean, median, std, IQR
* skewness, kurtosis
* entropy, energy
* min/max, p10/p90

### B. Texture features (within ROIs)

Using something like PyRadiomics (within segmented ROIs):

* **GLCM**: contrast, correlation, homogeneity, ASM, dissimilarity, entropy
* **GLRLM**: short-run emphasis, long-run emphasis, run-length nonuniformity
* **GLSZM**: small/large area emphasis, zone nonuniformity
* **NGTDM**: coarseness, busyness, complexity
* **GLDM**: dependence nonuniformity, etc.

**Recommendation:** Start with **a handful of ROIs (hippocampus L/R, entorhinal L/R, temporal GM)** and keep features to a manageable set (or do PCA).

---

## 6) Longitudinal feature engineering (key for survival)

You have repeated scans per subject. Even with tabular XGBoost, you can extract “trajectory” features that carry progression info.

You can do this in two ways:

### Option A: One row per subject (baseline + slopes)

For each ROI/measure (hippocampus volume, ventricle volume, AD signature thickness, etc.) compute:

* `value_at_baseline`
* `value_at_last_followup_before_event_or_censor`
* `absolute_change = last - baseline`
* `percent_change = (last - baseline) / baseline`
* `annualized_slope` (fit linear regression vs time in years)
* `slope_se` or residual std (trajectory noise)
* `min`, `max`, `mean` over follow-up

Also add:

* `n_scans_used`
* `followup_duration_years`

This is often the cleanest baseline for survival with 405 subjects.

### Option B: Time-dependent covariates (one row per scan interval)

Create “start-stop” rows per subject:

* interval: `[t_i, t_{i+1}]`
* features at `t_i` (or change from baseline to `t_i`)
* event flag = 1 if conversion happens at end of interval

This better matches longitudinal survival theory, but it’s more bookkeeping and not every XGBoost survival setup handles start-stop cleanly. Still, it’s a strong framework if you implement it carefully.

---

## 7) Change-map / deformation features (powerful if you register longitudinally)

If you do within-subject registration (follow-up → baseline), you can compute deformation-based summaries:

* **Jacobian determinant** stats in ROIs:

  * mean log-Jacobian in hippocampus, temporal lobe, ventricles
  * captures local tissue shrinkage/expansion
* Difference-image stats in ROIs (after intensity normalization)

These can be extremely predictive for progression because they encode atrophy patterns directly.

---

## 8) “Solid minimal baseline feature set” I’d start with (to avoid overfitting)

Given only ~405 subjects, I’d start with something like:

### Per-subject baseline + slopes (maybe 30–80 columns total)

**Time/coverage**

* `n_scans`, `followup_years`

**Scanner/protocol controls (baseline scan)**

* `Manufacturer`, `Model`, `FieldStrength`
* `voxel_size_x/y/z`, `TR`, `TE`, `TI`, `FlipAngle` (whatever exists)

**QC**

* `SNR`, `CNR`, `EFC` (or your simpler QC equivalents)

**Morphometry (baseline + annualized slope)**

* `TIV`
* hippocampus L/R (normalized) + slope
* entorhinal L/R thickness or volume + slope
* ventricles total (normalized) + slope
* posterior cingulate / precuneus / inferior parietal thickness (or volume) + slope
* mean cortical thickness + slope
* asymmetry indices (baseline and/or slope)

That baseline is usually already strong and interpretable, and you can later add radiomics or deformation features if needed.

---

## Small but important leakage note (for survival)

When you compute slopes/last values, make sure you only use scans **up to the event time** for converters, and up to **last follow-up** for censored subjects. Otherwise you accidentally let “post-conversion” anatomy influence the features.

---

If you want, I can propose a **column schema** (exact column names + data types), and a **tiered extraction plan** (Tier 1: header+JSON+QC, Tier 2: volumes/thickness, Tier 3: radiomics, Tier 4: deformation/Jacobians) that’s optimized for “get a baseline model running fast without painting yourself into a leakage corner.”
