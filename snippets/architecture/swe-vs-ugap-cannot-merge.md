---
title: "SWE vs UGAP: Why They Cannot Be Merged"
category: architecture
tags: [swe, ugap, divergence, mode-gate, frame-buffer, roi]
difficulty: intermediate
---

SWE and UGAP share the same UI shell but have semantically incompatible internals — the divergence is intentional and domain-driven, not a refactoring opportunity.

## Structure

| Aspect | SWE | UGAP |
|--------|-----|------|
| Mode gate | `ESMain::InElasto()` | `ESMain::IsInUGAPMode()` |
| AutoPos key | empty during Freeze | `"UGAPAcqAssistAnalyze"` during Measure |
| Frame buffer | `m_currentFrame_BS` (1 channel) | `m_currentFrame_BS` + `m_currentFrame_BS_UGAP` (2 channels) |
| ROI result type | `SWEROIProcessor::ProcessResult` | `ACResults` |
| `updateROICursorPosition` | throttles cursor position | transfers metadata + validates tilt angle |

## Why It Exists

The two modalities happen to display in the same region of the screen, but their physical measurement models, data pipelines, and state machines are different. A "unified" handler would require so many `if (isSWE) / if (isUGAP)` branches that it would be harder to maintain than two separate classes — which is exactly the problem the current architecture avoids.

## Key Points

- Never create a single unified handler — the divergence is the correct model, not a bug
- The shared UI shell is just visual; the underlying data and state are not shared
- Two-channel frame buffer in UGAP (`m_currentFrame_BS_UGAP`) has no SWE equivalent — any merge attempt must account for this
- `updateROICursorPosition` has the same name in both but different contracts — treat them as unrelated functions that happen to share a name
