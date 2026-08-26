# SWIM-360 Dynamic Camera Setup Computer Vision Swimmer Pose Estimation Dataset

COCO-17 keypoint annotations for swimmer pose estimation, captured from a
dynamic lane camera setup (phased swim, non-segmented strokes), below
water **ONLY**.

## This is a two-part dataset:

| | Contains | Where |
|---|---|---|
| **This repo** | Annotation registry (`data/`), the pre-computed train/valid/test split (`dataset_config.yaml`), and the scripts you need to use the dataset | GitHub |
| **Images** | Face-blurred (anonymised) frames | [Hugging Face](https://huggingface.co/datasets/UM-SWIM-360/swim-360-dynamic-cv-dataset) |

Images are hosted separately because they're large binary files; this repo
stays lightweight and diffable (Git-friendly JSON only). The two are matched
by file path — `data/<swimmer>/<camera_angle>/<stroke>/annotations/<video>/frame_NNNNNN.json`
pairs with `<video>/frame_NNNNNN.png` on Hugging Face.

## Quickstart

```bash
git clone https://github.com/UM-SWIM-360/swim-360-dynamic-cv-dataset.git
cd swim-360-dynamic-cv-dataset
python -m venv venv
venv\scripts\activate
pip install -r requirements.txt


# 1. Fetch the images into a folder of your choosing
python fetch_from_hf.py --repo_id UM-SWIM-360/swim-360-dynamic-cv-dataset --local_dir [name_of_hf_directory]

# 2. Compile the registry + images into a standard COCO dataset
python compile_coco.py --registry_root . --dataset_config dataset_config.yaml `
    --output_dir dataset --images_root "[name_of_hf_directory]\blurred_images"
```

That's it — `dataset_config.yaml` is already included in the repo (the
canonical published train/valid/test split), so step 2 is all you need.
This produces `dataset/{train,valid,test}/{images/, _annotations.coco.json}`,
ready to point standard COCO training pipelines at directly.

## Using run_pipeleine.py
Instead of running scripts individually, 'run_pipeline.py' runs all the necessary scripts in order to generate the COCO dataset from the [Hugging Face](https://huggingface.co/datasets/UM-SWIM-360/swim-360-dynamic-cv-dataset) blurred images.
```bash
python run_pipeline.py
```
## Potential Issues with 'fetch_from_hf.py'
If an error 11001 arises at any point during the execution of 'fetch_from_hf.py', stop the execution and delete the local directory that stored the static images.
This error occurs due to not being logged in to your huggingface account, limiting the amount of images you can download. In the case that you are not logged in, use the following commands

In the virtual environment:
```bash
python -m pip install --upgrade huggingface_hub
```
Login:
```bash
hf auth login
```
After running the login command, follow the instructions on screen and run:
```bash
python fetch_from_hf.py --repo_id UM-SWIM-360/swim360-static-cv-dataset --local_dir [name_of_hf_directory]
```
**OR**
```bash
python run_pipeline.py
```

### Issue: `xet` backend rate limit (429) during dataset download

**Symptom:**
```
ConnectionError: Network error: Request error: HTTP status client error (429 Too Many Requests), domain: https://huggingface.co/api/datasets/UM-SWIM-360/swim-360-dynamic-cv-dataset/xet-read-token/...
```
Occurs partway through `fetch_from_hf.py` (`snapshot_download`), often after downloading most of the dataset (e.g. 14.8/14.9GB).

**Cause:** Hugging Face's `xet` storage accelerator rate-limits aggressively on datasets with many files, independent of normal Hub rate limits.

**Fix:** Disable the `xet` backend before running the pipeline, falling back to the standard HTTP resolver:
```powershell
$env:HF_HUB_DISABLE_XET = "1"
python run_pipeline.py
```
`snapshot_download` with `local_dir` skips already-downloaded files, so re-running resumes rather than restarting.

---

### Issue: `504 Gateway Timeout` on individual file downloads

**Symptom:**
```
huggingface_hub.errors.HfHubHTTPError: Server error '504 Gateway Timeout' for url 'https://huggingface.co/datasets/UM-SWIM-360/swim-360-dynamic-cv-dataset/resolve/.../blurred_images/P010/Under Water/backstroke/.../frame_000110.png'
```
Occurs on a single file mid-download (with `HF_HUB_DISABLE_XET=1` set).

**Cause:** Transient CDN timeout on Hugging Face's side for one specific file request — not a rate limit, not a local config issue. With ~18k files being fetched, occasional one-off failures are expected.

**Fix:** Simply re-run the pipeline; `snapshot_download` resumes from where it left off:
```powershell
python run_pipeline.py
```
If timeouts recur frequently, increase the per-request timeout:
```powershell
$env:HF_HUB_DOWNLOAD_TIMEOUT = "60"
```
## Dataset conventions

- **Swimmer ID**: `P<number>` (e.g. `P007`)
- **Camera angle**: `Above Water` / `Under Water`
- **Stroke**: `freestyle`, `backstroke`, `breaststroke`, `butterfly`
- **Swim Segment**: `START`, `SWIM1`, `TURN`, `SWIM2`,`FINISH`
- **Keypoints**: standard COCO-17 layout (nose, eyes, ears, shoulders,
  elbows, wrists, hips, knees, ankles)

## Repository structure

```
data/                     # Annotation registry (this repo's core content)
  P007/Under Water/freestyle/
    metadata.json         # categories, resolution, per-video stats
    annotations/<video_stem>/
      frame_000000.json   # one micro-JSON per annotated frame
dataset_config.yaml       # The published swimmer -> train/valid/test split

fetch_hf.py               # Download the image set from Hugging Face
compile_coco.py           # Merge registry + images into standard COCO JSON
generate_split_config.py  # compute your own custom split

```

Everything above is what you need to *use* the dataset. Scripts used to
*build* it in the first place (pairing raw CVAT exports with source videos,
assembling the registry, face-blurring, uploading to Hugging Face) aren't
part of this repo — they're maintainer-side tooling, not needed to train on
the dataset.


# if you want a custom train/valid/test ratio for your own experiments:

```bash
python generate_split_config.py --registry_root . --output dataset_config.yaml \
    --train_ratio 0.6 --valid_ratio 0.2 --test_ratio 0.2 --force
```

`--force` is required since `dataset_config.yaml` already exists — this
overwrites it with a freshly-computed (still participant-disjoint) split.

---
# Dynamic Dataset — Metrics Summary

**Total frames:** 18,434 | **Total annotations:** 18,465 | **Image resolution:** 3840×2160 (100% of frames, all UHD) ([Hugging Face](https://huggingface.co/datasets/UM-SWIM-360/swim360-dynamic-cv-dataset))

---

## 1. Dataset Overview

| Metric | Value |
|---|---|
| Total frames | 18,434 |
| Total annotations | 18,465 |
| Total source videos | 46 |
| Participants (swimmers) | 23 |
| Strokes covered | 4 (backstroke, breaststroke, butterfly, freestyle) |
| Camera views | 1 (Under Water only — no above-water footage) |
| Phase coverage | 3 (swim1, swim2, turn) |

---

## 2. Coverage by Swimmer

![alt text](https://github.com/UM-SWIM-360/swim-360-dynamic-cv-dataset/blob/main/assets/images/distributions/frames_by_swimmer.png?raw=true)

![alt text](https://github.com/UM-SWIM-360/swim-360-dynamic-cv-dataset/blob/main/assets/images/distributions/videos_by_swimmer.png?raw=true)

---

## 3. Coverage by Stroke

![alt text](https://github.com/UM-SWIM-360/swim-360-dynamic-cv-dataset/blob/main/assets/images/distributions/frames_by_stroke.png?raw=true)

---

## 4. Coverage by Camera View

| Camera  | Videos |
|---|---|
| Under Water | 46 |

The dynamic dataset contains **no above-water footage at all** — every source video is Under Water, unlike the static dataset which mixed both views.

![alt text](https://github.com/UM-SWIM-360/swim-360-dynamic-cv-dataset/blob/main/assets/images/distributions/frames_by_camera.png?raw=true)

---

## 5. Annotation Density (swimmers labeled per frame)

![alt text](https://github.com/UM-SWIM-360/swim-360-dynamic-cv-dataset/blob/main/assets/images/distributions/annotations_per_frame.png?raw=true)

---

## 6. Keypoint Completeness per Annotation

![alt text](https://github.com/UM-SWIM-360/swim-360-dynamic-cv-dataset/blob/main/assets/images/distributions/keypoints_per_annotation.png?raw=true)

---

## 7. Keypoint Visibility Breakdown

Percentages are the share of all 18,465 annotations where the keypoint is confidently visible (v=2). Unlike the static dataset's breakdown, the dynamic metrics output only reports this visible share — occluded (v=1) vs. not-labeled (v=0) counts aren't split out separately here.

| Keypoint | Visible (v=2) |
|---|---|
| Left Ear | 8.7% |
| Right Ear | 12.6% |
| Left Eye | 21.1% |
| Right Eye | 23.3% |
| Nose | 38.1% |
| Right Shoulder | 42.9% |
| Left Shoulder | 44.5% |
| Right Elbow | 48.8% |
| Right Wrist | 52.8% |
| Left Elbow | 55.5% |
| Right Hip | 57.4% |
| Left Hip | 58.5% |
| Left Wrist | 59.8% |
| Left Knee | 72.2% |
| Right Knee | 73.6% |
| Left Ankle | 74.1% |
| Right Ankle | 75.9% |

![alt text](https://github.com/UM-SWIM-360/swim-360-dynamic-cv-dataset/blob/main/assets/images/distributions/keypoint_visibility.png?raw=true)

**Notes:**
- **Facial keypoints (eyes, ears) remain the least visible**, same as in the static dataset — left ear visible in only 8.7% of annotations, right ear 12.6%.
- **Lower body (hips/knees/ankles) is again far more visible than upper body**, consistent with the static dataset's pattern (57–76% vs. well under 60% for everything above the hips).
- **The left/right asymmetry doesn't hold in one consistent direction across all joints.** Ears, eyes, knees, and ankles are all more visible on the right side (matching the static dataset's pattern), but shoulders, elbows, and wrists are all more visible on the **left** side here — the opposite of static. Worth checking rather than assuming the same camera-side bias explanation carries over.

---

## 8. Participant × Stroke Coverage Gaps

![alt text](https://github.com/UM-SWIM-360/swim-360-dynamic-cv-dataset/blob/main/assets/images/distributions/coverage_heatmap.png?raw=true)

---

## 9. Participant × Camera × Stroke × Phase Coverage Gaps

Of 276 possible (swimmer × camera × stroke × phase) combinations, **46 are covered and 230 are gaps (83.3%)**.

Because this dataset has only one camera view (Under Water), camera view isn't a confound here the way it was for the static dataset — the gaps are entirely across swimmer × stroke × phase.

A few swimmers appear responsible for a disproportionate share of the gaps: **P002, P003, and P004** show zero frames across nearly every stroke/phase combination in the gap list, which tracks with their low total frame counts (326, 372, and 123 frames respectively — among the lowest of the 23 swimmers). **P011** (109 frames, the lowest of any swimmer) likely contributes heavily too, though it doesn't appear in the specific gap entries printed.

The full gap list (230 entries) wasn't fully enumerated in the metrics output used here — only a partial sample was shown before truncating ("...and 200 more") — so treat the above as a coverage-rate summary rather than a complete per-swimmer gap pattern. A full breakdown would need the untruncated 230-entry list.
