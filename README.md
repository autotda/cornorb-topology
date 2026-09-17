# CornOrb Topology

Research code for cubical persistent homology and multi-view classification of keratoconus from **rendered corneal topography**. The repository organizes the original CornOrb experiments into an installable Python package. It is not a clinical diagnostic application.

The image model implemented here is a shared **ConvNeXt-Tiny encoder with Transformer token fusion**, optionally using clinical and TDA tokens. DINOv2 was considered during planning but is not the released encoder.

## What Is Included

- CornOrb metadata cleaning, duplicate/conflict auditing, and patient-grouped splits.
- Three repeats of five outer folds, with training-side inner model selection.
- Six modality configurations: image, clinical, TDA, and their combinations.
- Clinical-lite and rendered-map mask/crop sensitivity analyses.
- Strict-CV stress testing, metrics-only output, resume support, and progress/ETA.
- HomCloud representative localization and historical GUDHI localization.
- A separate Kaggle training/test replication workflow and suspect-score analysis.
- Fold summary CIs and paired Wilcoxon comparisons with Holm adjustment.

Datasets, patient-level manifests, fitted features, trained weights, predictions, manuscript drafts, and historical experiment outputs are **not included**.

## Layout

```text
cornorb-topology/
  src/
    orbscan_experiment/          # Original dataset, models, training and TDA implementation
    cornorb_topology/
      cli.py                    # Lazy unified command-line entry point
      validation.py             # Patient/eye and nested-split audits
      environment.py            # Dependency version diagnostics
      workflows/
        data/
        features/
        training/
        stress/
        sensitivity/
        interpretability/
        analysis/
        legacy/                 # Historical fixed-split workflows, not primary strict CV
  configs/                      # Portable example presets; not recovered run configurations
  docs/                         # Protocol, reproducibility boundaries and release checks
  requirements/                 # Historical version observations, not an install lockfile
  tests/                        # Dependency-free contract and split tests
  .github/workflows/ci.yml
  CITATION.cff
  pyproject.toml
```

## Installation

Use a **new environment**, rather than changing the environment used for completed experiments. Python 3.11 is the suggested clean-install starting point. Install a matching PyTorch/torchvision pair for your CPU or CUDA installation, then install this package:

```bash
conda create -n cornorb-topology python=3.11
conda activate cornorb-topology
python -m pip install --upgrade pip
python -m pip install -e ".[tda]"
```

For HomCloud/GUDHI representative workflows:

```bash
python -m pip install -e ".[representatives]"
```

HomCloud optimal-cycle/volume routines may need platform-specific solver support. A successful package import does not establish that every optimization routine is available.

```bash
cornorb --help
cornorb doctor
```

The conservative installation constraints use NumPy 1.x and scikit-learn 1.3.2 to respect giotto-tda 0.6.2's declared requirements. The historical workstation had a different combination; see [environment notes](docs/ENVIRONMENT.md). A clean installation and GPU training have not been certified across all supported platforms. The included GitHub Actions workflow tests source/CLI/split contracts, not full training or dependency installation.

## Data

Download datasets separately and follow their original license and access conditions:

- [CornOrb on Zenodo](https://zenodo.org/records/17127265).
- [Keratoconus Detection on Kaggle](https://www.kaggle.com/datasets/elmehdi12/keratoconus-detection).

Place CornOrb under a local directory such as:

```text
data/cornorb/
  clinical_data_and_labels.csv
  ORBSCAN_Dataset/ORBSCAN_Dataset/
    <patient_code>/<eye>/*_Anterior.png
    <patient_code>/<eye>/*_Posterior.png
    <patient_code>/<eye>/*_Axial.png
    <patient_code>/<eye>/*_Pachymetry.png
```

An alternative extraction layout can be supplied with `--image-root`. Generated manifests contain local image paths: regenerate them on a new machine rather than publishing them or copying stale workstation paths. Do not commit datasets or patient-level files.

## Primary Workflow

Run commands from the repository root or supply explicit workspace paths. A preset's relative paths are interpreted from the **current working directory**, not from the JSON file's directory. Boolean `true` adds a flag; `false`/`null` omit it. Explicit command-line options are passed after preset options.

First inspect commands without importing scientific dependencies or starting training:

```bash
cornorb --config configs/cornorb_prepare.example.json --dry-run
cornorb --config configs/ablations.example.json --dry-run
```

Prepare and audit local splits:

```bash
cornorb prepare-cornorb --dataset-root data/cornorb
cornorb validate-splits --splits-dir experiment_setup/splits
```

Manifest generation refuses to overwrite an existing split directory. Use a new output directory for an alternative protocol. Downstream paths must then be updated explicitly.

Extract TDA features and run the six-way primary evaluation:

```bash
cornorb --config configs/tda.example.json
cornorb --config configs/ablations.example.json
cornorb statistics --runs-root runs/ablation_suite_v1
cornorb clinical-lite
cornorb mask-crop --help
```

**Important:** CornOrb's historical TDA extraction fits some diagram vectorizers on the entire extraction batch. Fold-wise feature scaling does not remove that transductive preprocessing. These workflows preserve the historical implementation, not a claim that every fitted preprocessing step is nested. Read [protocol boundaries](docs/PROTOCOL.md) before interpreting strict-CV results or changing this behavior.

Main experiments retain checkpoints needed by subsequent image stress evaluation. Run stress testing only after the corresponding clean folds are complete:

```bash
cornorb --config configs/stress_cv.example.json
cornorb stress-tables
```

The stress runner defaults to `metrics-only`: task metrics, compact metadata and summaries are kept, while bulky transient artifacts are removed where supported. Combined training/evaluation tasks can still temporarily need a checkpoint. Progress and ETA are printed and recorded in the stress output's `progress.json`. Add `--resume-existing` when resuming an interrupted run.

The example stress preset uses the four historical primary models. For causal modality attribution, explicitly include `image_clinical` and `image_tda` in `--models`; eligible models depend on the stress family. Do not interpret image-branch degradation with intact clinical/TDA features as a joint degradation of every modality.

For full-cohort representative statistics without per-case figure storage:

```bash
cornorb --config configs/representatives.example.json
```

## Independent-Source Replication

```bash
cornorb prepare-kaggle --dataset-root data/kaggle
cornorb extract-kaggle-tda
cornorb kaggle-replication
```

Models are trained and selected on the Kaggle train/validation source, then evaluated on its independent test source. This is **not** direct external testing of CornOrb-trained models. Suspect cases are excluded from binary fitting and assessed separately. Kaggle case identifiers are not verified longitudinal patient identifiers.

The public defaults use `replication` and `suspect_score` terminology. Historical output directories using `external_validation` or `suspect_risk` names can still be read by supplying their paths explicitly; those older names do not strengthen the validation claim.

## Testing

The basic tests need only Python and the source path. They do not access either dataset:

```bash
# Bash
PYTHONPATH=src python -m unittest discover -s tests -v
```

```powershell
# PowerShell
$env:PYTHONPATH = (Resolve-Path src).Path
python -m unittest discover -s tests -v
```

Optional scientific integration checks require installed scientific dependencies:

```bash
python -m unittest discover -s integration_tests -v
```

Only synthetic images/features are used; no pretrained weights are downloaded and no full training run is launched.

## License and Citation

Use `CITATION.cff` for the software title and author metadata. No paper DOI, acceptance status or external benchmark ranking is asserted.

The project code and original repository documentation are released under the [MIT License](LICENSE). This license does not cover the CornOrb or Kaggle datasets, pretrained weights, journal templates, or third-party software. Check [release requirements](docs/RELEASE_CHECKLIST.md), [migration notes](docs/MIGRATION.md), [verification notes](docs/VERIFICATION.md), and [GitHub metadata](docs/GITHUB_METADATA.md) before uploading.
