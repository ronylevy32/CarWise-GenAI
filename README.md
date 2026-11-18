# CarWise

**A visual guide to physical controls and dashboard warning lights in an unfamiliar car**

Generative AI course project | Rony Levy, Noa Cohen, Lian Maor

> **Status:** Project proposal (`2023 Kia Picanto` pilot). Dataset collection is in progress; no experiments have been run yet.

## Research questions

1. **Visual adaptation:** Does LoRA fine-tuning improve the recognition of physical controls and dashboard warning lights compared to a frozen vision-language baseline?
2. **Evidence grounding:** Does retrieving passages from the matching owner's manual edition improve the operating instructions and warning guidance a driver receives?

## Overview

Drivers in rental, borrowed, or newly purchased vehicles often encounter unfamiliar buttons or warning lights without knowing the exact terminology to search a printed manual. CarWise lets a parked driver select a component in a photo and ask a question in their own words.

Using the vehicle profile (**2023 Kia Picanto** pilot; planned expansion to **Hyundai, Toyota, and Volkswagen**), the system:
* **Identifies** the selected climate/audio control or dashboard warning indicator.
* **Grounds** its response in the matching owner's manual edition (`Kia JA 2023`) with an exact section citation.
* **Abstains** and asks for clarification whenever the manual edition is uncertain or retrieved passages conflict or lack sufficient evidence.

| Driver’s question | Selected region | Expected grounded response |
| :--- | :--- | :--- |
| “How do I clear the windshield?” | Front defogger button (`FRONT`) | Press the front windshield defogger button to clear condensation from the windshield. *(Kia JA 2023 reference: Windshield defrosting and defogging)* |
| “What does this red oil-can icon mean?” | Red oil-can warning light | Engine oil pressure is too low. Stop in a safe place immediately and contact your Kia dealer. *(Kia JA 2023 reference: Warning and indicator lights)* |

## System pipeline

```text
[Photo + Selected Region + Query] ──▶ [1. Vehicle Profile: 2023 Kia Picanto]
                                                     │
[Grounded Answer + Section Citation] ◀── [4. Qwen2.5-3B Answer & Abstention]
                 ▲                                   ▲
                 │                                   │
   [3. Hybrid Retrieval: BM25 + E5] ◀── [2. Visual Recognition: Qwen2.5-VL + LoRA/OCR]
```

1. **Confirm vehicle profile:** Match make, model, year, and market (`2023 Kia Picanto`) to filter retrieval to the exact manual edition.
2. **Recognize component:** Classify the selected region using **Qwen2.5-VL-3B-Instruct**, evaluating frozen vs. LoRA-adapted weights and comparing recognition with and without **Tesseract OCR** for printed labels (`FRONT`, `A/C`).
3. **Retrieve evidence:** Combine **BM25** lexical search with **multilingual E5-base** semantic search to retrieve the top-3 relevant manual passages.
4. **Generate & verify answer:** Synthesize a concise, cited response via **Qwen2.5-3B-Instruct**, withholding action steps when evidence is insufficient.

### Experimental design & evaluation targets

All four variants are evaluated on the exact same held-out test set alongside an **OpenCV template-matching** baseline (with a frozen CLIP + linear classifier CPU fallback if needed):

| Variant | Visual recognizer | Visual training | Manual retrieval | Research test |
| :--- | :--- | :--- | :--- | :--- |
| **A** | Qwen2.5-VL-3B-Instruct | Frozen | Off | Zero-shot baseline |
| **B** | Qwen2.5-VL-3B-Instruct | Frozen | On | **B vs. A:** Effect of manual retrieval |
| **C** | Qwen2.5-VL-3B-Instruct | LoRA-adapted | Off | **C vs. A:** Effect of visual LoRA tuning |
| **D** | Qwen2.5-VL-3B-Instruct | LoRA-adapted | On | **D vs. A:** Full CarWise pipeline |

* **Metrics:** Per-class F1 & **Macro-F1** (recognition); **Recall@1**, **Recall@3**, and **MRR** (retrieval); blinded two-reviewer scoring for instruction accuracy, citation support, and abstention; median & 95th-percentile mobile latency.
* **Pilot targets:** Variant **D** aims for at least **+10 percentage points** (+3 fully correct, cited answers across 30 test cases) over **A**, **≥80%** coverage on answerable cases, and **zero unsafe recommendations**.

## Datasets & annotation

| Dataset / Source | Role | Planned size & scope |
| :--- | :--- | :--- |
| **Kia Picanto 2023 Control Photos** | Climate & audio control recognition (`front defogger`, `A/C`, `volume`) | **75 real photos** |
| **Kia Picanto 2023 Dashboard Cases** | Warning recognition (`oil pressure`, `charging`, `ABS`, `check engine`, plus no-warning negatives) | **75 real cases** |
| **2023 Kia Picanto Owner’s Manual (`Kia JA 2023`)** | Hybrid retrieval corpus & ground-truth reference sections | Official manual indexed with section metadata |

* **Splits & quality control:** **90 training / 30 validation / 30 test** real photos (15 controls, 15 dashboard cases in test), grouped by capture session to prevent leakage. Two annotators independently label regions, classes, and manual sections (with a third resolving disagreements). Duplicates are removed, and faces, license plates, and VINs are masked.

## Team members and contributions

* **Lian Maor:** Leads dataset collection, anonymization, and annotation protocol.
* **Rony Levy:** Leads visual component recognition (Qwen2.5-VL-3B, LoRA fine-tuning, OCR integration, and OpenCV/CLIP baselines).
* **Noa Cohen:** Leads hybrid manual retrieval (BM25 + E5-base indexing), citation grounding, abstention logic, and evaluation.

## Repository structure

```text
slides/    Proposal presentation (PPTX and PDF)
```

Implementation code (`src/`), data preparation scripts (`data/`), experiment configs (`configs/`), and evaluation results (`results/`) will be added as the project progresses.
