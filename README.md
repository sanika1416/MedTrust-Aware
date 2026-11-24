# MedTrust Aware

**Trust-aware, explainable chest X-ray analysis research prototype**

MedTrust combines a multi-label chest X-ray classifier with analysis of
calibration, uncertainty, image quality, out-of-distribution (OOD) risk, and
similar reference cases. It returns an analysis through a Python HTTP API and a
prebuilt web interface.

> **Research use only.** This software is not a medical device and is not
> validated for clinical diagnosis or treatment. It must not be used as a
> substitute for review by a qualified clinician.

## Problem statement

A classification score alone does not describe whether an image is suitable
for analysis, whether a prediction is uncertain, or whether the input resembles
the data used by the model. MedTrust is an experimental effort to present
prediction and supporting reliability signals together, so that results can
be reviewed with more context.

## Key features

- Multi-label chest X-ray classification using DenseNet-121 and 14 pathology
  outputs.
- Per-class thresholds and temperature-based calibration.
- Technical image-quality analysis.
- OOD screening using maximum softmax probability, energy, and Mahalanobis
  scores.
- Monte Carlo dropout uncertainty estimates.
- Grad-CAM explanations and explanation-reliability analysis.
- Similarity retrieval over precomputed image embeddings, with structured
  reference-case evidence.
- A trust-reasoning result that combines analysis signals.
- HTTP API routes for health, authentication, analyses, and case history, with
  SQLite persistence.
- A prebuilt single-page web interface served by the Python server.

## Technology stack

| Area | Technologies present in this project |
| --- | --- |
| Model and analysis | Python, PyTorch, torchvision, NumPy, SciPy, pandas, scikit-learn, Pillow, Matplotlib, OpenCV |
| API and persistence | Python standard-library threaded HTTP server, SQLite |
| Web interface | Prebuilt React single-page application bundle |
| Containerization | Docker and Docker Compose |
| Tests | Python `unittest`-compatible tests; the checked-in suite also runs with pytest |

The checkout contains the compiled frontend bundle but **not** its original
frontend source, npm manifest, or lockfile. Consequently, rebuilding or
developing the frontend with npm is not currently supported from this checkout.

## Architecture

```text
Chest X-ray upload
       |
       v
Python HTTP API and input validation
       |
       v
Preprocessing -> DenseNet-121 -> calibrated multi-label scores
                                      |
       +------------------------------+------------------------------+
       |                 |                |             |            |
       v                 v                v             v            v
  Image quality       Uncertainty       OOD checks    Grad-CAM    Similarity
       +------------------------------+------------------------------+
                                      |
                                      v
                          Trust-aware analysis response
                             |                |
                             v                v
                      SQLite case store   Web interface
```

The main code areas are:

- `backend/`: HTTP server, inference orchestration, database, and management
  commands.
- `src/model.py`, `src/dataset.py`, and `src/transformer.py`: classifier,
  output labels, and image preprocessing.
- `src/calibration/`, `src/evaluation/`, `src/OOD/`, and
  `src/uncertainty/`: calibration, thresholds, OOD scoring, and uncertainty.
- `src/explainability/` and `src/quality/`: Grad-CAM/reliability and technical
  image-quality analysis.
- `src/similarity/` and `src/similarity_v2/`: embedding-based reference
  retrieval and evidence reporting.
- `frontend/dist/`: compiled web application assets.
- `tests/`: backend, OOD, and similarity tests.

## Data, model artifacts, and privacy

This public-source preparation intentionally does not distribute medical
images, datasets, model weights, generated evaluation artifacts, reference
embeddings/reports, or the local application database.

The existing project documentation names NIH ChestX-ray14, but this checkout
does not provide a complete dataset license/provenance record for the local
images, derived files, or model weights. Some local image files have EXIF
metadata, and the local SQLite database contains user/session and case-record
fields. Those files are excluded rather than assumed safe to publish. The
reference-image notes describe some examples as synthetic, but that statement
does not establish the provenance of every image in the project.

To run inference, obtain authorized model and derived artifacts separately and
place them at the paths expected by the code, including:

- `checkpoints/best_model.pt`
- `outputs/evaluation/thresholds.json`
- `outputs/calibration/temperature.json`
- `outputs/ood/artifacts/ood_thresholds.json`
- `outputs/ood/artifacts/mahalanobis_statistics.npz`
- The similarity index, manifest, metadata, and reference reports under
  `outputs/similarity_v2/`

Do not put patient data, unlicensed images, credentials, or private artifacts
in this repository. No project-level `LICENSE` file is present; no permission
to reuse the project or third-party assets should be inferred.

## Installation and setup

### Requirements

- Python 3.11 or 3.12 (the versions documented by the project).
- A compatible PyTorch installation. CPU inference is possible; GPU use
  depends on a matching PyTorch/CUDA environment.
- The separately obtained model and analysis artifacts listed above for
  inference.

### Install Python dependencies

From the repository root:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

On Linux or macOS, activate the environment with:

```bash
source .venv/bin/activate
```

If using an NVIDIA GPU, install the PyTorch build that matches the host's CUDA
setup before installing the remaining requirements.

### Configure and initialize

Configuration is read from process environment variables. `.env.example` is a
reference template; the Python server does not load a `.env` file by itself.
Docker Compose passes its declared settings to the container.

For a local PowerShell session, for example:

```powershell
$env:PORT = "8000"
$env:MEDTRUST_DB_PATH = "backend/medtrust.db"
python backend/manage.py init-db
python backend/manage.py migrate
python backend/manage.py check-readiness
```

The readiness command checks the required artifacts. It will report missing
artifacts until the authorized files have been supplied locally.

Database initialization does not create a shared default login. Register an
account through the application or provision an administrator interactively:

```powershell
python backend/manage.py create-admin --username admin@example.com --name "Project Administrator"
```

The command prompts for a password when run in an interactive terminal. Do not
put real passwords in shell history or commit them to the repository.

## How to run

After installing dependencies and placing the authorized artifacts at their
expected paths:

```powershell
python backend/server.py
```

Open [http://localhost:8000](http://localhost:8000). The live health endpoint
is `/api/health/live`; the API also provides versioned routes under `/api/v1/`.
Use `python backend/manage.py --help` to see the available database,
administrator, and readiness commands.

The Dockerfile builds the server and prebuilt frontend. Docker Compose mounts
the local model and output directories at runtime; these assets are not baked
into the image or included in the repository. Inference readiness still
requires the authorized artifacts.

## Tests

From the repository root:

```powershell
python -m unittest discover -s tests -p "test_*.py" -v
```

The suite passed in the inspected working copy (36 tests). Several tests
depend on the local model, output artifacts, and sample images, which are
intentionally not included in this repository. A fresh clone therefore needs
authorized test fixtures before the full suite can pass.

## Screenshots

No reviewed screenshots are included yet. Add screenshots only after checking
that they contain no patient information, identifiable image details, local
paths, or credentials.

## Future improvements

- Publish the frontend source and its dependency manifest/lockfile.
- Record authoritative licenses and provenance for every model, dataset,
  image, and derived artifact before distributing any of them.
- Add a CI workflow that runs tests with safe, synthetic fixtures and supported
  Python versions.
- Provide a documented, privacy-safe way to obtain or regenerate the required
  model and analysis artifacts.
- Add reviewed screenshots and a project license selected by the owner.
