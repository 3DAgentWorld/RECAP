# RECAP

## Building 3D Semantic Scene Graphs with Relational Evidence Capture and Pooling

[Overview](#overview) · [Method](#method) · [Results](#results) · [Qualitative Comparisons](#qualitative-comparisons) · [Physical-Robot Evaluation](#physical-robot-evaluation)

### Abstract

We present RECAP, a framework for constructing persistent 3D semantic scene graphs that support natural-language object queries and robot navigation. Frame-wise predictions can represent the same object as multiple nodes and produce inconsistent relations, while repeated viewpoints can reinforce these errors. To obtain open-vocabulary scene observations, we fine-tune a compact vision–language model using structured pseudo-labels generated offline by a larger model. After mapping observations into 3D, Entity Consolidation (L1) uses learned cross-frame associations to form persistent entities while retaining view-specific object names. Relation Belief Fusion (L2) scores relations between these entities using the same frozen compact model, then aggregates multi-view predictions with learned reliability weights while discounting repeated viewpoints. Experiments on 3DSSG and zero-shot ReplicaSSG show higher relation recall than FROSS and DeWorldSG. With RGB-D inputs and ground-truth poses, the relative gains over DeWorldSG are 27.55% and 18.25% on the two datasets, respectively. Controlled ablations show reduced entity fragmentation and improved target selection for relational queries. Downstream evaluation examines query-selected goals through shortest-path endpoints on a ground-truth navigation mesh and physical-robot navigation through historical image goals in previously scanned environments.

## Overview

![RECAP overview: offline supervision, compact observations, spatial lifting, entity consolidation, relation fusion, and language-based target selection.](https://github.com/3DAgentWorld/RECAP/releases/download/paper-assets/figure1-overview.png)

Offline supervision adapts the compact observer and trains L1/L2. During mapping, RGB keyframes provide object names, boxes, and relation descriptions; masks and geometry place these observations in 3D. L1 consolidates observations into persistent entities, and L2 aggregates relation evidence at those endpoints. Language queries then select entities for goal localization or physical-robot execution. Observation and relation scoring share frozen compact-model weights.

## Method

### Compact Open-Vocabulary Observations

RECAP samples one frame out of every five. Qwen3-VL-2B-Instruct predicts object and relation records in a structured schema after supervised fine-tuning on annotations generated offline by a 30B VLM. SAM3 masks restrict spatial lifting to object pixels. For RGB mapping, VGGT-Ω estimates geometry, with metric scale recovered through alignment to Depth Anything 3 predictions; the RGB-D setting uses sensor depth and benchmark-provided poses.

### Persistent Relational Memory

| Entity Consolidation (L1) | Relation Belief Fusion (L2) |
|:---:|:---:|
| ![L1 associates spatial observations with persistent entities and preserves observed object names.](https://github.com/3DAgentWorld/RECAP/releases/download/paper-assets/figure2-l1.png) | ![L2 selects geometrically eligible entity pairs and fuses per-view relation scores with reliability weighting and view-density discounting.](https://github.com/3DAgentWorld/RECAP/releases/download/paper-assets/figure2-l2.png) |

**Entity Consolidation (L1).** Geometric, semantic, quality, temporal, and relational-context evidence support cross-frame association. Compact objects are assigned jointly, while local plane alignment and footprint connectivity handle structural surfaces. Persistent entities retain distinct observed names for language matching.

**Relation Belief Fusion (L2).** Geometry and joint visibility select ordered entity pairs. An additional call to the shared, frozen observation model scores each pair over a task-specific predicate set that includes `NONE`. Learned reliability weights and view-density discounting control each observation's contribution to a weighted log-opinion pool.

L1 is supervised with 3DSSG instance identities; L2 uses directed relation annotations. Validation-selected settings are frozen for testing and transfer. Object names remain open vocabulary, while relation scoring uses a fixed task-specific codebook.

### Language-Based Target Selection

Object queries use MiniLM to match the query against each entity's retained names. Relational queries additionally combine subject and anchor matches with L2's stored predicate probability. The selected entity supplies a spatial goal for NavMesh endpoint evaluation or a historical image goal for the physical robot.

## Results

### Overall Scene-Graph Performance

Recall and mean recall are reported as percentages. The RGB variants use estimated depth and poses; RGB-D rows use sensor depth and ground-truth poses. FROSS and DeWorldSG results are quoted from their papers under the same benchmark evaluation protocol. ReplicaSSG is evaluated zero-shot.

| Dataset | Method | Mapping Input | Relation Recall | Object Recall | Predicate Recall | Object mRecall | Predicate mRecall |
|:---|:---|:---|---:|---:|---:|---:|---:|
| 3DSSG | FROSS | RGB-D | 27.90 | 62.40 | 33.00 | 63.80 | 18.00 |
| 3DSSG | DeWorldSG | RGB-D | 50.20 | 75.00 | 57.30 | 70.60 | 36.00 |
| 3DSSG | RECAP | RGB | 58.84 | 68.78 | 65.37 | 67.30 | 52.40 |
| 3DSSG | RECAP | RGB-D | **64.03** | 74.42 | **70.60** | **75.20** | **56.50** |
| ReplicaSSG | FROSS | RGB-D | 22.30 | 26.10 | 27.80 | 28.80 | 20.40 |
| ReplicaSSG | DeWorldSG | RGB-D | 38.20 | 35.40 | 45.30 | 38.10 | 27.30 |
| ReplicaSSG | RECAP | RGB | 41.46 | 48.12 | 47.08 | 48.30 | 38.21 |
| ReplicaSSG | RECAP | RGB-D | **45.17** | **51.42** | **50.08** | **52.29** | **43.88** |

### Controlled Mechanism Studies

The observation audit covers 8,339 object-answerable images, including 8,256 relation-answerable images. Empty-output rates are computed on answerable frames, and all values use final outputs after the shared format-repair and retry procedure.

| Observation Model | Object Empty-Output Rate | Relation Empty-Output Rate | Object Precision | Relation Precision |
|:---|---:|---:|---:|---:|
| Base 2B | 20.10% | 10.70% | 98.10% | 88.20% |
| + non-empty prompting | 0.00% | 10.40% | 97.50% | 85.90% |
| + structured SFT | 0.00% | 0.00% | 97.70% | 92.40% |

For L1, compared methods share observations and semantic readout. Fragmentation counts predicted entities per represented ground-truth instance; its ideal value is 1. False merge measures parents containing multiple ground-truth identities. Correct instance@1 measures object-query target selection.

| Entity Memory | Compact Fragmentation | Surface Fragmentation | False Merge | Correct Instance@1 |
|:---|---:|---:|---:|---:|
| Shared observations + FROSS-style association | 2.83 | 13.17 | 3.11% | 59.23% |
| L1 without surface continuity | 1.50 | 12.33 | 4.59% | 65.74% |
| Full L1 | 1.54 | 3.14 | 4.04% | 69.17% |

For L2, compared methods share full-L1 endpoints, the frozen scorer, per-view predictions, and geometry-only candidates. Presence F1 distinguishes a listed relation from `NONE`; relational target@1 measures correct subject selection. Memory metrics use 3DSSG, and target-accuracy metrics use the Replica query sets; both report unweighted scene means.

| Edge Aggregation | Predicate mRecall | Presence F1 | Relational Target@1 |
|:---|---:|---:|---:|
| Majority vote | 45.15% | 62.10% | 31.84% |
| Unweighted log pooling | 50.70% | 64.58% | 32.86% |
| + reliability weighting | 54.29% | 67.96% | 36.27% |
| + view-density discounting (full L2) | 56.50% | 71.14% | 38.79% |

### Relational Goal Localization

On all 11 Replica test scenes, 265 relational queries yield 1,325 query–start trials. The predicted scene graph selects the goal; a shared ground-truth NavMesh supplies traversable space and ideal shortest paths. SR@1m measures whether the path endpoint lies within 1 m Euclidean distance of the intended object's ground-truth 3D bounding box. Results are averaged equally across scenes. This is shortest-path endpoint evaluation, not action-level closed-loop navigation.

| Persistent Memory | Endpoint SR@1m |
|:---|---:|
| Legacy association + majority relations | 23.01% |
| L1 + majority relations | 28.07% |
| Full RECAP | 33.65% |
| Full RECAP + closed-vocabulary name projection | 18.59% |

## Qualitative Comparisons

### Entity Consolidation on Replica

![Replica entity comparison: FROSS retains a bed-corner fragment labeled pillow; RECAP consolidates observations into a persistent bed entity.](https://github.com/3DAgentWorld/RECAP/releases/download/paper-assets/figure3-entity-comparison.png)

FROSS retains a bed-corner fragment labeled “pillow” (top). Its same-class association rule leaves the differently named observation separate from the bed. RECAP combines geometric, semantic, and temporal evidence to consolidate bed observations into a persistent entity (bottom).

### Relation Belief Fusion on Replica

![Replica relation comparison with missed and spurious relations highlighted for FROSS on the left and RECAP on the right.](https://github.com/3DAgentWorld/RECAP/releases/download/paper-assets/figure4-relation-comparison.png)

FROSS (left) exhibits spurious ceiling-light links and missing table–chair relations. RECAP (right) rescores geometrically eligible pairs, including pairs without an initial relation statement. Reliability weighting and view-density discounting limit poor and repeated evidence in the persistent relation graph.

## Physical-Robot Evaluation

![One physical query episode: query and candidate objects, selected persistent entity and image goal, and the measured robot endpoint.](https://github.com/3DAgentWorld/RECAP/releases/download/paper-assets/figure5-physical-robot.png)

The evaluation covers **8 indoor environments, 80 distinct object-finding queries, and 160 attempted episodes**. A teleoperated RGB scan builds the scene memory and a topological image map. MiniLM matches query phrases to retained entity names, with L2 additionally disambiguating relational descriptions. The selected entity provides a historical image goal for a frozen ViNT navigator. Depth observations provide obstacle protection during execution.

| Metric | Successful Episodes | Rate |
|:---|---:|---:|
| Correct-instance selection | 112 / 160 | 70.00% |
| Physical SR@1m | 93 / 160 | 58.13% |
| Joint correct-instance selection and SR@1m | 87 / 160 | 54.38% |

Physical SR@1m requires no execution failure and a laser-measured distance of at most 1 m from the robot's front-edge center to the intended object's nearest measurable surface. Rejections, unavailable image goals, collisions, timeouts, and manual takeovers remain in the denominator. The three panels above show one episode, whose measured endpoint distance is **0.72 m**.

---

Project page maintained by [Ziran Yin](https://github.com/uprightderran).
