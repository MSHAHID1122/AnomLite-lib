1. Data Ingestion (data/raw/)
Read DICOM files

DICOM = Digital Imaging and Communications in Medicine.

Each file contains both pixel data (the actual X-ray image) and metadata (patient ID, acquisition settings, timestamps).

Load each file into an in-memory array (e.g., a NumPy array) plus a metadata dictionary.

Store untouched copies

As soon as you read the file, write a byte-for-byte copy into data/raw/.

This ensures you can always go back to the exact original, which is critical for reproducibility and regulatory audits.

2. Data Cleaning & Quality Control (data/processed/)
Reject corrupt or incomplete files

Check for read errors, missing pixel arrays, or metadata fields.

Move any bad files to data/processed/rejected/ with a short log of why.

Normalize orientations

Some X-rays are stored upside-down or mirrored.

Use metadata tags (e.g., PatientOrientation) or simple image heuristics (rib patterns) to flip/rotate into a consistent frontal view.

Anonymize metadata

Strip out any Protected Health Information (names, dates of birth, IDs).

Replace with randomly generated study IDs if needed.

3. Preprocessing (preprocessing/)
Resizing & Rescaling

Resize every image to a fixed resolution (e.g., 512×512 pixels).

Rescale pixel intensities to a standard range, often [0, 1] or [0, 255].

Intensity Normalization

Min–max scaling:

𝐼
′
=
𝐼
−
𝐼
min
⁡
𝐼
max
⁡
−
𝐼
min
⁡
I 
′
 = 
I 
max
​
 −I 
min
​
 
I−I 
min
​
 
​
 
Z-score normalization:

𝐼
′
=
𝐼
−
𝜇
𝜎
I 
′
 = 
σ
I−μ
​
 
You can compute μ, σ per-image or across the entire “normal” training set.

Denoising (optional)

Gaussian blur: smooths out high-frequency noise but can blur edges.

Median filter: preserves edges better, good for salt-and-pepper noise.

Contrast Enhancement

Histogram equalization: spreads out the most frequent intensity values.

CLAHE (Contrast Limited Adaptive HE): applies histogram equalization in small regions to avoid over-amplifying noise.

4. Patch Extraction (optional, preprocessing/patch_extractor.py)
Divide into patches

Slide a small window (e.g., 32×32 px) across the image, either overlapping or non-overlapping.

Save patches

Organize into folders by class (e.g., data/processed/patches/normal/ and eventually …/anomaly/).

This localizes your analysis and can yield finer anomaly maps.

5. Feature Engineering
Flatten raw pixels

Turn each image (or patch) into a 1D vector, ready for classical methods.

Texture descriptors

Histogram of Oriented Gradients (HOG)

Gray Level Co-occurrence Matrix (GLCM) features: capture local texture patterns.

Edge maps or filter responses

Apply Sobel, Scharr, or filter banks (e.g., Gabor) to emphasize edges and fine structures.

6. Model Training (chest_xray_anomaly/detectors/)
Classical detectors on “normal” training data only:

PCA-based outlier detection: fit a low-dimensional subspace, large reconstruction error ⇒ anomaly.

One-Class SVM (OCSVM): learns a boundary around normal data in feature space.

Texture models: build statistical models of texture features.

Patch-based detectors: train small autoencoders or distance-based detectors on patches.

7. Anomaly Scoring
Per-image or per-patch score

Reconstruction error (e.g., pixel-wise MSE for autoencoders).

Distance to the learned normal manifold (PCA residuals, OCSVM decision function).

Anomaly maps (heatmaps)

Reshape patch scores or pixel reconstructions back into image space.

Generate a heatmap highlighting high-score (suspicious) areas.

8. Thresholding & Decision (utils/metrics.py)
Determine optimal thresholds

Use a held-out validation set of normal and anomalous images.

For example, pick a cutoff at the 95th percentile of the normal error distribution.

Classify

Image-level: above threshold ⇒ “anomalous.”

Pixel-level: heatmap values above threshold ⇒ anomalous regions.

9. Visualization & Reporting (utils/visualization.py)
Overlay heatmaps

Superimpose the anomaly map on the original X-ray (e.g., transparent red overlay).

Summary plots

ROC curves to show trade-off between true/false positive rates.

Score histograms comparing normal vs. anomalous images.

10. Storage of Results
Structured outputs

e.g., results/phase1/images/, …/heatmaps/, …/metrics/.

Logging & configs

Save processing parameters (resizing, filters), threshold values, and model hyperparameters alongside results.

This end-to-end traceability is essential for audits and future re-runs.