# FDE Multimodal Preprocessing Skill

Version: `fde-preprocess/1.0`

In one sentence: process multimodal raw data into comparable, traceable standard records, and hand them to the main cleaning workflow under a fixed contract.

## Scope

Preprocessing only: three-layer storage (raw layer read-only and faithful, evidence layer holding locations and intermediate products, result layer holding only standard records), per-modality standardization, cross-modal alignment (time, space, semantics), and a preprocessing quality gate.

It does NOT: adjudicate conflicts into final facts, turn dedup candidates into final dedup conclusions or delete data, decide whether anomalies should be removed, certify structured extraction as facts, run the main cleaning acceptance regression, or handle scheduling, ingestion, version release, or report generation.

## Usage

The full specification is in `SKILL.en.md` (English) and `SKILL.md` (Chinese). Trigger: when multimodal raw data needs a unified format, evidence pointers, and time, space, and semantic alignment for downstream cleaning, use this skill.

Iron rules: after every data read, submit a confirmation card first; do not enter precise processing unless pointer integrity is 100%; mark broken pointers as `broken`, block them from the result layer, and never silently repair them; grey zones output only candidates, evidence, and suggestions — never bind or delete on your own.

## Threshold policy

Any threshold not given in the document is marked to be calibrated; any number given is marked as an example starting value, for cold start only, and must not be rewritten as a general standard. Before project use, calibrate with your own annotated samples: use 100 samples to find workflow problems, then expand to 1000 for acceptance after fixes.

## Files

- `SKILL.md`: full specification (Chinese)
- `SKILL.en.md`: full specification (English)
- `README.md`: this note (Chinese)
- `README.en.md`: this note (English)
