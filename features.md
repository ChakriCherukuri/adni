## 1) Final flattened column schema (one row per subject; ~405 rows)

Below is a **single “subjects” table schema** that’s (a) leakage-safe for survival, (b) small enough to work with 405 subjects, and (c) extendable later.

### A. IDs, cohort bookkeeping, follow-up

* `subject_id` (string)
* `n_visits_total` (int) — total scans found
* `n_visits_used` (int) — scans used for features (up to event/censor; see leakage rule below)
* `followup_years` (float) — from baseline MCI to last scan
* `manufacturer_mode` (string) — most common across used visits
* `model_mode` (string) — most common across used visits
* `field_strength_mode_t` (float) — most common across used visits
* `site_mode` (string) — most common across used visits (if available)

### B. Survival label columns (what you train on)

* `event_observed` (int) — 1 if converts MCI→AD during follow-up, else 0
* `event_time_years` (float) — time from baseline MCI to first AD (if event) else to last scan (censor)
* `mci_bl_datetime` (datetime) — baseline MCI date/time
* `event_datetime` (datetime) — first AD date/time (null if censored)
* `censor_datetime` (datetime) — last available scan date/time

**Optional (for AFT survival objective)**

* `aft_y_lower` (float) — = `event_time_years` for all
* `aft_y_upper` (float) — = `event_time_years` if event else `+inf`

### C. Baseline protocol/geometry controls (from baseline MCI scan)

(JSON + NIfTI header)

* `meta_field_strength_t_bl` (float)
* `meta_tr_s_bl` (float)
* `meta_te_s_bl` (float)
* `meta_ti_s_bl` (float)
* `meta_flip_angle_deg_bl` (float)
* `hdr_dim_x_bl` (int)
* `hdr_dim_y_bl` (int)
* `hdr_dim_z_bl` (int)
* `hdr_vox_x_mm_bl` (float)
* `hdr_vox_y_mm_bl` (float)
* `hdr_vox_z_mm_bl` (float)
* `hdr_voxvol_mm3_bl` (float)
* `hdr_fov_x_mm_bl` (float)
* `hdr_fov_y_mm_bl` (float)
* `hdr_fov_z_mm_bl` (float)

### D. Baseline QC / intensity (from baseline MCI scan)

(Use MRIQC-style if you can; otherwise compute simple brain/background stats)

* `qc_brain_mask_vol_mm3_bl` (float)
* `qc_brain_mean_bl` (float)
* `qc_brain_std_bl` (float)
* `qc_brain_p01_bl` (float)
* `qc_brain_p50_bl` (float)
* `qc_brain_p99_bl` (float)
* `qc_bg_mean_bl` (float)
* `qc_bg_std_bl` (float)
* `qc_brain_bg_ratio_bl` (float)
* `qc_snr_bl` (float, optional)
* `qc_cnr_bl` (float, optional)
* `qc_efc_bl` (float, optional)
* `qc_cjv_bl` (float, optional)

### E. Baseline global morphometry (Tier 3+)

* `seg_tiv_mm3_bl` (float)
* `seg_gm_mm3_bl` (float)
* `seg_wm_mm3_bl` (float)
* `seg_csf_mm3_bl` (float)
* `seg_brain_mm3_bl` (float) — GM+WM
* `seg_bpf_bl` (float) — (GM+WM)/TIV
* `seg_ventricles_mm3_bl` (float)
* `seg_ventricles_norm_bl` (float) — ventricles/TIV

### F. Baseline AD-relevant ROI volumes/thickness (Tier 4/5+)

Pick **one** of these sets depending on what your parcellation produces:

**If you have volumes only**

* `roi_hippocampus_l_vol_norm_bl` (float)
* `roi_hippocampus_r_vol_norm_bl` (float)
* `roi_hippocampus_asym_bl` (float)
* `roi_amygdala_l_vol_norm_bl` (float)
* `roi_amygdala_r_vol_norm_bl` (float)
* `roi_entorhinal_l_vol_norm_bl` (float)
* `roi_entorhinal_r_vol_norm_bl` (float)
* `roi_parahippocampal_l_vol_norm_bl` (float)
* `roi_parahippocampal_r_vol_norm_bl` (float)
* `roi_inferior_temporal_l_vol_norm_bl` (float)
* `roi_inferior_temporal_r_vol_norm_bl` (float)
* `roi_middle_temporal_l_vol_norm_bl` (float)
* `roi_middle_temporal_r_vol_norm_bl` (float)
* `roi_fusiform_l_vol_norm_bl` (float)
* `roi_fusiform_r_vol_norm_bl` (float)
* `roi_precuneus_l_vol_norm_bl` (float)
* `roi_precuneus_r_vol_norm_bl` (float)
* `roi_posterior_cingulate_l_vol_norm_bl` (float)
* `roi_posterior_cingulate_r_vol_norm_bl` (float)
* `roi_inferior_parietal_l_vol_norm_bl` (float)
* `roi_inferior_parietal_r_vol_norm_bl` (float)
* `roi_lateral_ventricle_l_norm_bl` (float)
* `roi_lateral_ventricle_r_norm_bl` (float)
* `roi_third_ventricle_norm_bl` (float)

**If you have cortical thickness too**
Add:

* `roi_mean_cortical_thick_mm_bl` (float)
* `roi_entorhinal_l_thick_mm_bl` (float)
* `roi_entorhinal_r_thick_mm_bl` (float)
* `roi_inferior_temporal_l_thick_mm_bl` (float)
* `roi_inferior_temporal_r_thick_mm_bl` (float)
* `roi_middle_temporal_l_thick_mm_bl` (float)
* `roi_middle_temporal_r_thick_mm_bl` (float)
* `roi_fusiform_l_thick_mm_bl` (float)
* `roi_fusiform_r_thick_mm_bl` (float)
* `roi_precuneus_l_thick_mm_bl` (float)
* `roi_precuneus_r_thick_mm_bl` (float)
* `roi_posterior_cingulate_l_thick_mm_bl` (float)
* `roi_posterior_cingulate_r_thick_mm_bl` (float)
* `roi_inferior_parietal_l_thick_mm_bl` (float)
* `roi_inferior_parietal_r_thick_mm_bl` (float)
* `roi_ad_signature_thick_mm_bl` (float) — mean of a chosen AD-signature set

### G. Longitudinal change features (computed using scans up to event/censor)

For a **small biomarker set** `X ∈ {hippocampus_norm, entorhinal_thick (or volume), ventricles_norm, bpf, ad_signature_thick}` compute:

For each `X`, include:

* `long_<X>_last` (float)
* `long_<X>_delta` (float)
* `long_<X>_pctchg` (float)
* `long_<X>_slope_yr` (float)
* `long_<X>_mean` (float)
* `long_<X>_std` (float)
* `long_<X>_n_obs` (int)

Concrete recommended set (keeps features manageable):

* `long_hippocampus_norm_*` (use mean of L/R or include both separately)
* `long_entorhinal_*` (thickness if available, else volume norm)
* `long_ventricles_norm_*`
* `long_bpf_*`
* `long_ad_signature_thick_*` (if thickness available)

### Leakage rule (critical)

When building all `long_*` features:

* For **converters**: only use scans with `acq_datetime <= event_datetime` (choose `<=` or `<` and stick to it; I’d use `<= first AD scan` if you interpret that scan as “at conversion time”).
* For **censored**: use all scans up to `censor_datetime` (last scan).

This ensures you’re not using post-event information.

---

## 2) How do you create labels? Is this regression or classification?

### You are doing **survival analysis**

That means your “labels” are **(time, event)**:

* **`event_observed`**: did the subject convert from MCI → AD during observed follow-up?

  * `1` if they ever have an AD-labeled scan after baseline MCI
  * `0` if they never have AD during follow-up (censored)

* **`event_time_years`**: time from baseline MCI to:

  * first AD scan (if event_observed=1), else
  * last available scan (if event_observed=0)

That’s it. Those two columns define the survival target.

### Is it regression or classification?

It’s **neither in the usual sense**:

* **Not standard classification** because “stable MCI” isn’t truly negative forever — some are just not observed long enough (censoring).
* **Not standard regression** because the time is **censored** for non-converters (you don’t know true conversion time).

Survival learning handles this properly.

### What does the model output?

Depending on the survival method:

* Cox-style models output a **risk score** (relative hazard)
* Discrete-time hazard models output **probability of converting in each time bin**
* AFT models output a **distribution/estimate of time-to-event**

You can still turn it into a classification decision, e.g.:

> “Probability of conversion within 24 months > 0.5”

…but that’s derived from the survival output, not the training label.

---

### Minimal label-building algorithm (from your folder structure)

For each subject:

1. Sort visits by datetime folder name
2. Identify baseline:

   * first visit where label is MCI (dx_bin=0)
   * exclude subject if no MCI visit (for an “MCI→AD” study)
3. Find event:

   * first visit after baseline with label AD (dx_bin=1)
4. Set:

   * `event_observed = 1` if event exists else 0
   * `event_time_years = (event_datetime - baseline_datetime)/365.25` if observed
     else `(last_datetime - baseline_datetime)/365.25`

---

If you tell me whether you consider the **first AD-labeled scan** as the conversion time (common) or you want something like “midpoint between last MCI and first AD”, I can recommend the cleanest convention and how to keep it consistent across subjects.
