# Efficient 3D Character Animation Using Pose Estimation — A Resource-Constrained Approach

A ComfyUI pipeline that animates a still character image to follow the motion in a driving video, **engineered to run on consumer GPUs**. Instead of reaching for the heaviest models, it deliberately trades them for lighter, well-chosen alternatives and benchmarks the trade-offs:

- **OpenPose** for motion capture instead of the heavier **DWPose**
- **SD1.5** checkpoints instead of **SDXL**
- **LCM** (Latent Consistency) sampling to cut step counts
- **IPAdapter** to lock the character's identity across frames, **AnimateDiff** to generate the motion

The goal was a genuinely usable character-animation workflow for people *without* a datacenter GPU — and then to measure which model/step/LCM combination actually delivers the best quality-per-compute.

> **Provenance:** I designed and built this pipeline end-to-end. It was later written up as a group semester research paper at Chandigarh University (*Efficient 3D Character Animation Using Pose Estimation: A Resource-Constrained Approach* — co-authors Ishan Dev, Ch. Sai Viswanath Sarma, M. Soumith; faculty advisor Prof. Ruksana). The full draft is included as [`Research Paper Draft.docx`](Research%20Paper%20Draft.docx).
>
> It was also published on OpenArt's community workflow library, where it reached **160+ downloads** before OpenArt deprecated its public ComfyUI workflow library in 2026 — which is why this repository is now the canonical home for the workflows and assets. The original listing (`openart.ai/workflows/shepherd_buttery_70/effecient-3d-character-animation/iasYnsRpkR0x3R3ZTtJK`) now redirects to the OpenArt home page and is no longer accessible.

---

## How it works

The pipeline takes two inputs — a **character reference image** (the look) and a **driving video** (the motion) — and renders an animation of that character performing the motion.

| Stage | What happens |
|---|---|
| **1. Character input** | IPAdapter loads the reference image and prepares it (Prep for ClipVision) so the character's identity is injected into every generated frame. |
| **2. Driving video** | The motion video is loaded and sampled into frames. |
| **3. Pose estimation** | OpenPose (via ControlNet) extracts a pose skeleton from each frame — the lightweight stand-in for DWPose. |
| **4. Generation** | AnimateDiff, conditioned on the pose skeletons and the IPAdapter character embedding, generates the animated frames. |
| **5. Render & combine** | KSampler denoises, VAE decodes, and Video Combine stitches the frames into the final clip. |

### Pipeline walkthrough

**1 — IPAdapter: load and prep the character reference image**

![IPAdapter character input nodes](6.png)

**2 — Load the driving video and sample its frames**

![Load driving video nodes](1.png)

**3 — Extracted driving frames**

![Extracted video frames preview](2.png)

**4 — OpenPose ControlNet extracts the pose skeleton per frame**

![OpenPose ControlNet pose estimation](3.png)

**5 — AnimateDiff + IPAdapter generation nodes**

![AnimateDiff and IPAdapter nodes](4.png)

**6 — KSampler render → VAE decode → Video Combine output**

![KSampler render and Video Combine output](5.png)

---

## Results

The workflow was benchmarked across **eight configurations** — four motion modules, each run **with and without LCM** — to find the best quality/compute trade-off.

![Output comparison: non-LCM vs LCM across four motion modules](Output%20Comparison%20Chart.png)

**Winner: `v3sd15mm` + LCM (8 steps).** It delivered the best clarity, lighting, sharpness, and motion tracking while preserving the character's consistency. Without LCM, `v3sd15mm` suffered heavy background color bleeding; the 4-step AnimateDiff-Lightning module captured expressions but lost quality and consistency. `v3sd15mm` + LCM was the balanced sweet spot.

Every configuration's rendered clip is in [`Project Outputs/`](Project%20Outputs) and mapped to its workflow in the table below.

---

## The workflows

All eight benchmarked ComfyUI graphs are in [`Workflows/`](Workflows) — drop any `.json` into ComfyUI to run it. Each row links the workflow to its rendered output clip (configuration verified from the metadata embedded in each render).

| Sampling | Configuration | Workflow | Rendered clip |
|---|---|---|---|
| Non-LCM | AnimateDiff-Lightning, 4 steps | [`4step_animatediff_lightning.json`](Workflows/4step_animatediff_lightning.json) | [`AnimateDiff_00106.mp4`](Project%20Outputs/AnimateDiff_00106.mp4) |
| Non-LCM | AnimateDiff-Lightning, 8 steps | [`8step_animatediff_lightning.json`](Workflows/8step_animatediff_lightning.json) | [`AnimateDiff_00108.mp4`](Project%20Outputs/AnimateDiff_00108.mp4) |
| Non-LCM | mm-sd-v15-v2, 20 steps | [`20step_mmsdv15v2.json`](Workflows/20step_mmsdv15v2.json) | [`AnimateDiff_00112.mp4`](Project%20Outputs/AnimateDiff_00112.mp4) |
| Non-LCM | v3-sd15-mm, 20 steps | [`20step_v3sd15mm.json`](Workflows/20step_v3sd15mm.json) | [`AnimateDiff_00113.mp4`](Project%20Outputs/AnimateDiff_00113.mp4) |
| LCM | AnimateDiff-Lightning, 4 steps | [`4step_lcm_animatediff_lightning.json`](Workflows/4step_lcm_animatediff_lightning.json) | [`AnimateDiff_00107.mp4`](Project%20Outputs/AnimateDiff_00107.mp4) |
| LCM | AnimateDiff-Lightning, 8 steps | [`8step_lcm_animatediff_lightning.json`](Workflows/8step_lcm_animatediff_lightning.json) | [`AnimateDiff_00109.mp4`](Project%20Outputs/AnimateDiff_00109.mp4) |
| LCM | mm-sd-v15-v2, 8 steps | [`8step_lcm_mmsdv15v2.json`](Workflows/8step_lcm_mmsdv15v2.json) | [`AnimateDiff_00110.mp4`](Project%20Outputs/AnimateDiff_00110.mp4) |
| **LCM** | **v3-sd15-mm, 8 steps — best** | [`8step_lcm_v3sd15mm.json`](Workflows/8step_lcm_v3sd15mm.json) | [`AnimateDiff_00111.mp4`](Project%20Outputs/AnimateDiff_00111.mp4) |

---

## Tech stack

- **[ComfyUI](https://github.com/comfyanonymous/ComfyUI)** — node-based Stable Diffusion runtime
- **[AnimateDiff-Evolved](https://github.com/Kosinkadink/ComfyUI-AnimateDiff-Evolved)** — motion modules (`v3_sd15_mm`, `mm_sd_v15_v2`, AnimateDiff-Lightning)
- **ControlNet + OpenPose** — pose estimation / motion capture
- **[ComfyUI_IPAdapter_plus](https://github.com/cubiq/ComfyUI_IPAdapter_plus)** — character identity conditioning
- **WD14 Tagger** — automatic prompt tagging from the character image
- **EpicRealism (SD1.5)** — base checkpoint
- **VideoHelperSuite** — video load / frame sampling / Video Combine
- **LCM** — Latent Consistency sampling for low step counts

> While building this on an AMD/DirectML setup, I hit (and helped the maintainer debug) a KSampler-freeze bug in IPAdapter — testing fixes on AMD hardware he couldn't reproduce on. See [cubiq/ComfyUI_IPAdapter_plus#266](https://github.com/cubiq/ComfyUI_IPAdapter_plus/pull/266).

## Repository layout

```
Workflows/                   # 8 benchmarked ComfyUI workflow graphs (.json)
Project Outputs/             # Rendered animation clips + stills (AnimateDiff_00106–00113)
1.png … 6.png                # Pipeline node screenshots (walkthrough above)
Output Comparison Chart.png  # 8-configuration quality comparison
Research Paper Draft.docx    # Full semester research-paper write-up
```
