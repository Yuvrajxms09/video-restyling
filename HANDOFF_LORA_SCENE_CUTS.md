# Video Restyling Workflow Handoff

## Project focus

Improve MiniMax-H3 video-to-video restyling for videos with scene cuts, while preserving motion and mouth movement. The current best direction is `restyle_lora_wf` combined with OmniShotCut scene detection.

## Executive summary

The current evidence points to this conclusion:

- `restyle_lora_wf` with scene-cut processing produces better restyling quality than processing the entire video in one pass.
- `restyle_lora_wf` whole-video processing is more temporally consistent in some cases, but the style is weak and the result is not as good overall.
- `community_workflow` without LoRA gives the smoothest motion and the best mouth movement, but it performs poorly on videos with scene cuts.
- Applying LoRA to `community_workflow` is currently not a viable path: it reduces motion substantially and introduces artifacts.
- OmniShotCut is worth continuing for H3 scene-cut handling.
- Viggle is not worth pursuing for this restyling use case. It performs poorly even on simple videos without cuts and can reorganize videos with cuts.

## Evidence reviewed

- [`video1.mp4`](evidence/video1.mp4) — best current output: `restyle_lora_wf` with scene-cut detection, especially `shot_context`.

<video controls width="640" src="https://raw.githubusercontent.com/Yuvrajxms09/video-restyling/main/evidence/video1.mp4"></video>

- [`video2.mp4`](evidence/video2.mp4) — second-best direction: `restyle_lora_wf` processing the whole video, including a video that contains cuts. Motion is more coherent, but the style weakens later in the video.

<video controls width="640" src="https://raw.githubusercontent.com/Yuvrajxms09/video-restyling/main/evidence/video2.mp4"></video>

- [`video3.mp4`](evidence/video3.mp4) — mouth-motion reference baseline from `community_workflow`. It shows the strongest mouth movement and should be used to compare against `restyle_lora_wf`.

<video controls width="640" src="https://raw.githubusercontent.com/Yuvrajxms09/video-restyling/main/evidence/video3.mp4"></video>

Include `video1.mp4` and `video2.mp4` as the main quality comparison: scene-cut LoRA is the best overall result, and whole-video LoRA is the backup baseline with weaker style. Include `video3.mp4` only as the motion and mouth-movement reference. Do not present Viggle or LoRA-applied community-workflow results as candidate solutions; they were tested and produced poor restyling, reduced motion, and/or artifacts.

## Current ranking of approaches

### 1. `restyle_lora_wf` + OmniShotCut scene cuts

This is the leading approach for scene-cut videos. The video is split into detected shots, each shot receives its own first-frame Magic Hour style reference, each shot is run through H3, and the results are trimmed/concatenated back together.

The `shot_context` mode has generally produced the best results among the scene-cut modes. It gives H3 additional temporal context around a shot and cyclically wraps at EOF when required. `shot_exact` has produced worse visual results and should remain an experimental mode only.

### 2. `community_workflow` without LoRA

This currently gives the smoothest motion and strongest mouth movement. However, the single whole-video approach does not handle scene cuts reliably, and the scene-cut version is not currently strong enough for difficult videos.

### 3. `restyle_lora_wf` whole video

This is useful as a baseline. It can preserve overall continuity better than independently processed shots, but the restyling strength is weak and the output quality is below the LoRA scene-cut approach.

### 4. `community_workflow` with LoRA

Do not prioritize this path right now. Testing showed reduced motion and visible artifacts. It is substantially worse than `community_workflow` without LoRA.

### 5. Viggle

Do not prioritize. It is weak on simple videos without cuts, can damage videos that do not contain cuts, and may reorganize content when cuts are present.

## Workflows and supported modes

### `restyle_lora_wf`

The LoRA-based restyling workflow supports:

- `whole_video` — process the complete video in one inference pass.
- `shot_context` — detect scene cuts, process each shot with additional temporal context, then trim and concatenate the results. This has produced the best LoRA results so far.
- `shot_exact` — detect scene cuts and process each exact shot independently. This has produced worse results and is not the preferred mode.

For scene-cut videos, each detected shot receives a separately restyled first-frame reference before inference.

### `community_workflow`

The reference-guided restyling workflow can process a complete video or detected scene-cut shots. It currently gives smoother motion and better mouth movement than `restyle_lora_wf`, but its scene-cut results are weak on difficult videos. Enabling LoRA in this workflow is not currently recommended because of motion loss and artifacts.

### `restyle_lora_wf` scene-cut flow

```text
input video
  -> normalize to 24 fps
  -> OmniShotCut detects shot ranges
  -> create one source clip per shot/context window
  -> extract first frame from every planned clip
  -> restyle every first frame with Magic Hour
  -> upload each clip and its matching reference to ComfyUI
  -> run H3 LoRA independently per clip
  -> trim each generated clip to its requested shot duration
  -> concatenate clips in shot order
  -> restore original audio when configured
```

Before inference, the scene-cut mode extracts and restyles the first frame for each planned shot, then runs the corresponding shot through the workflow and joins the results in shot order.

## Problems to fix

`restyle_lora_wf` has unsmooth motion: movement can appear to pause or jump, with short freezes in the output video. In addition, mouth movement is present in `community_workflow` output but is little or absent in `restyle_lora_wf` output. Trace the LoRA workflow’s temporal conditioning, reference conditioning, prompt path, and audio/video inputs to determine why mouth motion is lost, then bring the stronger mouth movement into the LoRA workflow without sacrificing its better restyling quality.

## Important comparison with Sage

The strongest current contrast is:

```text
community_workflow without LoRA: smoother motion, better mouth movement, weaker scene-cut handling
restyle_lora_wf with scene cuts: better style/restyling on cut videos, unsmooth motion and weak mouth movement
```

The priority is therefore not to replace the LoRA workflow. The priority is to retain its stronger restyling while reducing its temporal damage.

## Recommended next experiments

Run experiments one variable at a time and keep the same source video, resolution, prompt, seed policy, shot ranges, and output muxing wherever possible.

1. Use `restyle_lora_wf` with `shot_context` as the primary scene-cut baseline.
2. Compare it with `restyle_lora_wf` in `whole_video` mode on the same videos and prompts.
3. Use `community_workflow` without LoRA as the motion and mouth-movement reference baseline.
4. Trace why mouth movement is stronger in `community_workflow` than in `restyle_lora_wf`.
5. Keep `shot_exact` as a diagnostic comparison only because it has produced poor outputs.

## Handoff goal

Improve `restyle_lora_wf` with OmniShotCut `shot_context` until it retains its stronger restyling quality while matching the smoother motion and mouth movement seen in `community_workflow`.
