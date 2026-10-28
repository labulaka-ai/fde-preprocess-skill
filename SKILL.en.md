# FDE Multimodal Preprocessing Skill

**Version:** `fde-preprocess/1.0`
**Purpose:** Turn multimodal raw data into comparable, traceable standard records, and hand them to the main cleaning workflow under a fixed contract.
**Operating principles:** Fidelity, evidence retention, rerunnability. Never treat a candidate as a conclusion. Never let model confidence stand in for evidence.

---

## 1. Trigger and Scope

### 1.1 When It Applies

Use this skill when the input contains one or more of the following kinds of data, and the data needs standardization, localization, alignment, or traceable candidates:

- Text: posts, documents, subtitles, OCR text, ASR transcripts.
- Images: still images, screenshots, charts, scanned pages.
- Audio: speech, interviews, narration, ambient sound.
- Video: video streams, clips, keyframes, subtitles, and audio tracks.
- Metadata: time, location, account, source, acquisition information, file inventory.

### 1.2 Where Preprocessing Stops

Preprocessing stops at **comparable and traceable**. Its output must provide:

1. Values standardized per modality.
2. Cross-modal time, space, and semantic associations backed by evidence.
3. Clear statements of data quality, coverage, and open items.
4. Lineage between raw values, intermediate results, and standard records.
5. Candidates and a quality report that the main cleaning workflow can take over.

**Out of scope for this skill:**

- Adjudicating conflicts and producing final facts.
- Reaching final deduplication conclusions or deleting data.
- Deciding whether anomalies should be removed.
- Producing final structured extraction and certifying it as fact.
- Acceptance regression for main cleaning.
- Scheduling jobs, ingestion, version release, report generation.

### 1.3 Three-Layer Storage Boundary

| Layer | Stores | Rules |
|---|---|---|
| Raw layer | Raw files, raw records, acquisition metadata | Read-only and faithful; never overwrite raw values. |
| Evidence layer | OCR/ASR results, detection boxes, keyframes, alignment results, candidate matches, quality diagnostics, intermediate products | Each item carries a source pointer, tool version, parameters, and run lineage; never dress a candidate up as a final conclusion. |
| Result layer | Standard records that conform to the unified event contract | Standard records only; content without an evidence pointer does not enter precise statistics and goes to the pending-review list only. |

A correction must never overwrite a raw value. Every correction leaves a trace: raw value, corrected value, evidence grade, reason, review status, model version, rule version, vote score, and threshold. When a field does not apply, fill it with `null` and state the reason. Never omit it in a way that makes "absent" indistinguishable from "not yet processed".

---

## 2. General Rules for AI Interaction

The AI is the executor that processes and organizes. It is not a substitute for the evidence chain. After every data read, submit a **confirmation card** first, then decide whether processing can proceed.

### 2.1 Hard Rules

1. Verify the data and pointers before doing precise processing.
2. If pointer integrity is below `100%`, do not enter precise processing. Output the confirmation card, the blocking items, and repair candidates first. Mark records with broken pointers as `broken` and block them from the result layer.
3. Do not silently repair broken pointers. You may propose a recovery candidate, but you must keep the original pointer, the candidate target, the matching basis, and the human review status. Never treat a candidate as already repaired.
4. Records without evidence pointers do not enter precise statistics. They go to the pending-review list only.
5. Follow this skill's fixed order. Do not skip the quality gate or bind candidates into events early.
6. In grey zones, output only candidates, evidence, and suggestions. Do not bind or delete on your own.
7. When a threshold is in a grey zone, evidence is insufficient, or several workable methods exist, spell out how the methods differ, their risks, and your suggestion, then wait for the owning main workflow to decide.
8. Write any threshold that has not been given as **to be calibrated**. Threshold examples supplied by the user are all marked as **example starting values, for cold start only**, and must not be rewritten as general standards.
9. Preprocessing may record baseline distributions for later drift detection. It must not declare drift on the basis of a baseline difference alone.
10. In outputs, keep "observed value", "algorithm candidate", and "reviewed value" distinct. Do not use wording that hides the evidence grade.

### 2.2 Grey-Zone Output Template

```text
Grey-zone ID:
Related source_id / evidence_pointer:
Problem type:
Current input and context:
Candidates:
  - Candidate A:
    Supporting evidence:
    Opposing or missing evidence:
    Method and version:
    Score and what it means:
  - Candidate B:
    Supporting evidence:
    Opposing or missing evidence:
    Method and version:
    Score and what it means:
Threshold status: example starting value / calibrated value / to be calibrated
Risks and impact:
Suggested next step:
Current status: pending review; not bound, not deleted
```

---

## 3. Post-Read Confirmation

### 3.1 Confirmation Card Template

After reading data, the first deliverable must be the confirmation card. Fields may be filled with `unknown` or `not computed`, but the reason must be stated.

```text
[Confirmation Card]
source_id:
Data batch / source description:
File inventory:
File hash algorithm and values:
  - File:
    Hash:
Modalities and counts:
  - Text:
  - Images:
  - Audio:
  - Video:
  - Metadata / other:
Time range (original timezone, normalized timezone):
Spatial coordinate system and coverage:
Total pointers:
Intact pointers:
Pointer integrity: intact / total = %
Broken pointer list:
Missing files / missing modalities list:
Damaged files list:
Pending issues (encoding, timezone, coordinate system):
Pending-review list:
Hash vs. source file consistency check:
Conclusion: precise processing allowed / precise processing blocked
Blocking reason:
```

**Calculation rules:**

- When total pointers are zero, record integrity as `not_applicable`. Never report it falsely as `100%`.
- When pointers exist, integrity is the number of pointers that parse and resolve to raw evidence, divided by total pointers.
- Any pointer that cannot be resolved or fails validation is marked `broken` and blocks precise processing. Do not rewrite pointers automatically through fuzzy matching.
- The file hash algorithm must be stated. If the algorithm or the byte range of the file is undetermined, mark it pending review. Hashes identify file versions; they do not substitute for source proof.
- The time range must state the original timezone and the normalized timezone after conversion. If the timezone is unknown, do not assume the local timezone.

### 3.2 Actions After the Confirmation Card

Once the confirmation card is done:

- `Precise processing allowed`: start with Section 4, in order.
- `Precise processing blocked`: only do inventory and fault localization that do not change the data, submit the pending-review items, and output no precise result-layer records.
- Precise processing may resume only after the repair plan has been confirmed by the main workflow, the evidence chain is complete, and the recomputed pointer integrity is `100%`.

---

## 4. Fixed Order of Operations

The following order is fixed. Do not reorder it.

1. **Inventory and confirmation**: produce the confirmation card, recording the file inventory, hashes, modality counts, and time and space ranges.
2. **Pointer validation**: validate each source pointer one by one; any `broken` pointer blocks precise processing.
3. **Preserve the raw layer**: confirm raw values are read-only, files are re-readable, and raw metadata is stored separately from content.
4. **Take a configuration snapshot**: record schema, model, rule, calibration set, thresholds, preprocessing parameters, and the run environment identifier; write to be calibrated for any parameter not specified.
5. **Standardize per modality**: process text, images, audio, video, and metadata separately first.
6. **Produce evidence-layer products**: save detections, recognitions, timestamps, boxes, keyframes, vectors, or candidate associations, keeping the tools and versions that produced them.
7. **Cross-modal alignment**: generate alignment candidates in time, space, then semantic order; every association must trace back to raw evidence.
8. **Quality gate check**: compute coverage, pointer completeness, model metrics, error budgets, and the grey-zone share; if the gate fails, block handover.
9. **Produce result-layer records**: write only standard records that conform to the unified event contract and carry evidence pointers. Candidates or records lacking evidence go to pending review.
10. **Hand over to main cleaning**: submit the standard interface, quality gate report, grey-zone list, configuration, and version matrix. Preprocessing does not adjudicate on behalf of main cleaning.

---

## 5. Method Selection

At every decision point, record the method used, the conditions for switching, and the cost. The default method is not a forced answer; data characteristics and validation results take priority.

### 5.1 Overall Selection Table

| Decision point | Recommended starting point | Applicable conditions | Conditions to switch methods or stop | Cost and risk |
|---|---|---|---|---|
| File identification | Confirm by magic number / parseable format, cross-checked against the extension | Format is decodable | Magic number conflicts with extension, or the decoder errors: isolate as damaged / pending review | Multi-format parsing adds runtime and compatibility cost; an extension alone cannot prove content is correct |
| Time standardization | Parse the explicit timezone and normalize to UTC, while keeping the original string and original timezone | Source provides a timezone or offset | Timezone unknown: keep the raw value, mark pending review, do not infer | Conversion hides differences in the original expression, so the source value must be kept |
| Coordinate standardization | Project under a stated coordinate system and record the conversion parameters | Coordinate system known, coordinates verifiable | Coordinate system missing, axis order unclear, or range anomalous: pending review | A wrong projection or axis order creates plausible-looking misalignment |
| Text duplication candidates | SimHash → MinHash → semantic vector check | Large text volume needing fast candidate retrieval | Short text, template text, or paraphrases: raise the review share; do not reach an automatic dedup conclusion | Hash approximation can miss paraphrases or wrongly merge template content |
| Image similarity candidates | dHash exact screening → pHash approximation → CLIP check | Image candidates need initial screening | Heavy cropping, small chart changes, semantically close but different subjects: human review, or a dedicated local-feature method, method to be calibrated | pHash is fairly robust to some transforms but does not guarantee the same subject; CLIP similarity is not the same as being the same image |
| Video similarity candidates | Compare keyframe fingerprint sequences after removing intro and outro | Video structure and frame extractability known | Major shot reordering, local splicing, or speed changes: candidates only, dedicated method to be calibrated | Sequence compression and frame extraction strategy affect matching results |
| OCR | Detect text regions, then recognize the text in each region | Text in the image is visible and regions are locatable | Handwriting, curved text, low resolution, or occlusion: evaluate by hard-case strata and pick a dedicated method | Recognition accuracy does not mean the box-to-content attribution is correct |
| ASR | Produce word-level timestamps, and evaluate text and numbers separately | There is a decodable audio track | Overlapping speech, music, severe noise: report failure / insufficient confidence; do not substitute whole-sentence timing for word-level timing | Word-level localization raises alignment cost; transcription accuracy does not mean the speaker is correct |
| Video frame extraction | Base-rate scan plus densification under pre-defined trigger conditions | Event times, subtitles, or target objects can trigger checks | Trigger scope unclear, or gradual transitions missed badly: raise coverage and run dedicated acceptance, parameters to be calibrated | Too-sparse extraction misses targets; too-dense extraction raises compute and review cost |

Any parameter in this table that is not specified by task data or user facts must be marked **to be calibrated**.

---

## 6. Module Specifications

Each module covers: the problem it solves, inputs, outputs, a checklist, failure modes, and an acceptance gate. Any acceptance gate threshold that has not been given is **to be calibrated**.

### 6.1 Raw Intake and Pointer Inventory

**Problem solved:** Make sure later results can be traced back to the raw file or raw record.

**Inputs:** Files, record inventory, source metadata, acquisition time, storage location information.

**Outputs:** File inventory, hashes, modality identification, pointer status, confirmation card.

**Checklist:**

- Files can be read in full; size, format, and decoding results are consistent.
- Hash algorithm, hash value, and the object hashed are stated.
- Each record can be located to a raw file, page, frame, time span, or coordinate range.
- File names, extensions, and actual content agree.
- Raw data is read-only; raw values are preserved.

**Failure modes:** Truncated files, unsupported encoders, inventory disagreeing with actual files, expired pointers, pointers that resolve only to a whole media file and not to the relevant segment.

**Acceptance gate:** Pointer integrity must be `100%` before precise processing; other file completeness and inventory thresholds are to be calibrated.

### 6.2 Unified Event Contract

**Problem solved:** Carry cross-modal event candidates in a consistent structure without losing source evidence.

**Unified event:**

```text
(
  subject,
  object,
  start_ms,
  end_ms,
  spatial_box,
  modality,
  content,
  evidence_pointer
)
```

Contract requirements:

- Time is expressed in milliseconds. Start and end are both relative to a stated time base; if the base is unknown, mark pending review and do not guess.
- Spatial box coordinates are normalized to `[0, 1]` under the declared media frame or canvas coordinate system. When a spatial box is meaningless, use `null` and note `not_applicable`.
- Modality takes the type matching the actual content. Multimodal associations are split into separately locatable events and linked through relationship fields.
- Content stores the standardized value and traces back to the raw value and evidence-layer output.
- Evidence pointers must resolve to objects and segments in the raw layer. Content without a pointer does not enter precise statistics; it goes to pending review only.
- When the subject or object cannot be determined reliably, fill in `unknown` or `null`. Never substitute an inferred value for an observed value.

**Version fields, at minimum:**

```text
schema_version
model_version
rule_version
calibration_version
config_version
source_id
evidence_pointer
run_id
```

`run_id` is the unique run identifier for this processing; its exact generation rule is to be determined. The project contract may specify the exact field names and schema serialization format; field semantics must not change.

**Correction audit-trail fields:**

```text
original_value
corrected_value
evidence_grade
reason
review_status
model_version
rule_version
vote_score
threshold
```

Any field that does not apply is expressed with an explicit empty value. Evidence grades A–E are **ordinal grades**, not probabilities.

### 6.3 Text Standardization and Duplication Candidates

**Problem solved:** Normalize text representation, locate text evidence, and produce duplication or near-duplication candidates.

**Inputs:** Raw text, encoding, source, language/script metadata, locatable record pointers.

**Full processing chain:**

1. Read the raw bytes and keep the original text; detect encoding anomalies without overwriting the source text.
2. Parse Unicode; keep the original string and record the normalized result as a derived value. The Unicode normalization form to use is to be calibrated.
3. Record language, script, and segmentation information. When automatic identification is unreliable, mark pending review.
4. When OCR/ASR text is needed, keep the evidence pointers and model information of the corresponding modality.
5. Compute character/word-level metrics or similarity candidates; keep the distance algorithm, parameters, and text length.
6. Output duplication candidates and similarity measures. Do not output "deleted" or final "identical" conclusions.

**Method selection:**

- **SimHash:** 64-bit, Hamming distance `≤3` as the strict starting point; 128-bit starting point `6–8`. It is a candidate screening starting point, not a duplication verdict.
- **MinHash:** Long-text Jaccard `0.8` as the empirical starting point; for short text under `50` characters, raise the threshold to `0.85–0.9` and add edit-distance or n-gram checks. The number of permutations is usually `128` or `256`.
- **Semantic vectors:** After hash candidates, check paraphrases or semantically close items; model and similarity threshold to be calibrated. Semantic closeness alone never proves duplication.
- When text is too short or too templated for hash features to be stable, raise human review or use local rules. Do not apply the long-text convention.

The numbers above are all **example starting values, for cold start only**.

**Checklist:** Original text recoverable; encoding anomalies visible; empty and whitespace-only values have a status; character length and tokenization convention recorded; each candidate keeps its distance and parameters.

**Failure modes:** Encoding misjudgment damaging characters; normalization erasing meaningful forms; short-text false matches; identical templates over-merged; near-synonym paraphrases missed.

**Acceptance gate:** Text parse success rate, encoding anomaly rate, character and word-level metrics, and the false-positive / false-negative budget for candidates must all be calibrated against a gold standard and error budgets. "High similarity" cannot substitute for acceptance.

### 6.4 OCR and Image-Text Spatial Attribution

**Problem solved:** Locate text in images and produce association candidates between text content and spatial regions.

**Inputs:** Images or video frames, raw image pointers, image size and orientation information.

**Full processing chain:**

1. Verify image decoding, orientation, size, color channels, and spatial pointers.
2. **Detect** text regions and produce region boxes; detection and recognition are two separate steps.
3. **Recognize** the text in each region, recording the text, region, model version, and confidence.
4. Keep the pre-detection original image and all transformation parameters; map coordinates back to the original image.
5. Compare OCR regions with target object or event boxes using IoU, containment ratio, and center distance.
6. Generate attribution candidates using time overlap and context; do not bind on box relationships alone.
7. Report OCR detection quality, recognition quality, and image-text attribution quality separately.

**Box attribution conventions:**

- IoU `0.3–0.4`: candidate only.
- IoU `>0.5`: strong evidence.
- Containment ratio `>0.8` and center distance below threshold: strong evidence.
- Center distance threshold to be calibrated.
- Strong evidence is **not direct binding**; time overlap and context confirmation are still required.

These values are **example starting values, for cold start only**.

**Metrics:** Evaluate detection and recognition separately; report CER (character error rate) and WER (word error rate) separately; also record box localization, spatial attribution, and misses separately. The detection metric calculation convention, target box IoU gate, and quality gate are to be calibrated.

**Hard cases:** Low resolution, rotated or perspective text, multi-column layouts, subtitles overlapping background text, occlusion, watermarks, vertical text or handwriting, subtitle change moments.

**Failure modes:** Missed boxes, inconsistent box merging/splitting, correctly recognized text attributed to the wrong object, coordinates not mapped back after preprocessing crops, video subtitles already changed but frame extraction missed them.

**Acceptance gate:** Set separate budgets for CER/WER, detection quality, box localization, and attribution correctness. Without a gold standard, do not claim quality targets have been met.

---

### 6.5 Audio, ASR, and Speakers

**Problem solved:** Transcribe speech into locatable text while keeping speaker and acoustic evidence boundaries clear.

**Inputs:** Audio or video audio tracks, time base, media metadata such as sample rate and channels.

**Full processing chain:**

1. Check that the audio decodes; save sample rate, channels, duration, audio track, and source pointer.
2. Record whether resampling, mixing, cropping, or denoising was applied, with parameters; the original audio stays unchanged.
3. Produce the transcript and require **word-level timestamps**; if only sentence-level results exist, mark them as not meeting precise alignment requirements.
4. Compute WER, CER, and number error rate separately. The text normalization convention for numbers must be fixed first and its version noted.
5. Do speaker separation only in multi-speaker scenarios; single-speaker narration does not get speaker separation.
6. In multi-speaker scenarios, evaluate DER separately and report hard segments such as overlapping speech and speaker changes.
7. Record hard cases stratified by accent type and SNR band, including Cantonese and Sichuan dialect; for example, the band below `10 dB` gets its own column. Other SNR cut points are to be calibrated.
8. When aligning acoustic events with video, subtitle, or OCR time candidates, keep each original timestamp and the offset evidence.

**Metrics and errors:**

- Word-level timestamps are mandatory.
- Report WER and CER separately; list number error rate on its own.
- DER is accepted only for multi-speaker scenarios.
- Report accent type and SNR strata separately; do not let hard segments hide inside an overall average.
- Scoring conventions for tokenization, number normalization, silent segments, and overlapping speech are to be calibrated.

**Failure modes:** Sentence-level timestamps passed off as word-level; noisy segments dropped from transcription; number transcription errors diluted by ordinary WER; two overlapping speakers merged into one; language/dialect misidentified; video audio time base offset.

**Acceptance gate:** WER, CER, number error rate, word-level time localization, and multi-speaker DER are each evaluated against their error budgets. Thresholds that have not been calibrated must not be written as established standards.

### 6.6 Video Frame Extraction and Keyframes

**Problem solved:** Produce frame evidence sufficient for event localization and cross-modal verification, at controlled cost.

**Inputs:** Decodable video, duration, frame rate or readable timestamps, pre-declared trigger conditions.

**Full processing chain:**

1. Read duration, frame rate, and frame timestamps; for variable frame rate, locate by actual timestamps and do not assume a constant frame interval.
2. Define the base rate and the densified rate; the exact parameters must be calibrated.
3. **Write trigger conditions in advance**: scene cuts, subtitle changes, target objects, ASR target words, and similar.
4. For each trigger interval, record start and end times, the trigger reason, and the source evidence.
5. Set an upper bound on a single densification, for example at most `30 seconds`; when exceeded, split into multiple intervals or block, with the exact boundary policy to be calibrated.
6. Record frame extraction time, original frame number/timestamp, transformations, and image pointers.
7. Separately sample gradual transitions; they are a high-miss area and must be accepted separately.
8. When frame extraction coverage is insufficient, do not claim "the object does not appear in the video".

**Cost formula:**

\[
\text{total frames}
=
\text{base rate}\times\text{total duration}
+
(\text{densified rate}-\text{base rate})\times\text{total trigger duration}
\]

Units must agree: frame rate in frames per second, duration in seconds.

**Worked example (full inputs, calculation, and output):**

- Inputs: video `60 seconds`; base rate `1 frame/second`; two trigger intervals totaling `16 seconds`; densified rate `5 frames/second`.
- Base-rate scan: `1 × 60 = 60 frames`.
- Extra frames in trigger intervals: `(5 − 1) × 16 = 64 frames`.
- Total: `60 + 64 = 124 frames`.
- Full frame-by-frame at `30 fps`: `30 × 60 = 1800 frames`.
- Output: plan to extract `124 frames`, plus the two trigger intervals, trigger types, the actual number of frames extracted, and the dedicated acceptance result for gradual-transition scenes. The planned count is not the actual successfully decoded count; missing frames must be reported.

The rates above are the inputs of this worked example, not general recommended values.

**Failure modes:** Trigger conditions written after the fact; gradual scene cuts missing key changes; frame rate treated as constant when the video is variable frame rate; a missed frame extraction treated as the event not existing; densification without an upper bound causing cost to run away.

**Acceptance gate:** Frame extraction coverage, trigger hit rate, frame pointer integrity, gradual-transition miss metrics, and cost budget are to be calibrated; samples must include gradual-transition cases.

### 6.7 Cross-Modal Alignment: Time, Space, Semantics

**Problem solved:** Associate observations from different modalities into event candidates without hiding uncertainty.

**Inputs:** OCR regions and times, ASR word timestamps, frame timestamps, spatial boxes, text/object candidates, and evidence pointers.

**Order:**

1. **Time**: first unify the time base, original timezone, and media offsets; if missing or unclear, mark pending review.
2. **Space**: then unify canvas, orientation, coordinate system, and normalized boxes; keep a trace of parameters before and after conversion.
3. **Semantics**: last, align subjects, objects, words, and visible content, producing candidates and supporting evidence.
4. Output the three kinds of evidence, the algorithms, and the threshold versions for each alignment; whenever one kind is unreliable, state exactly what is missing.

**Dynamic time tolerance:**

- Do not use a hardcoded `500 ms` as a single window for all situations.
- Set it from frame rate as `3–5` times the frame interval. At `30 fps`, one frame is about `33 ms`, corresponding to about `100–167 ms`.
- Example conventions per scenario: opening narration `200–300 ms`, interviews about `500 ms`.
- A one-sided `500 ms` window spans a total of `1 second`, about `30 frames`, which may be too wide for fast-paced narration.
- **Tolerance must be smaller than the minimum interval between adjacent events**; if not satisfied, do not attribute automatically. Narrow the tolerance or move to pending review.
- Event types, the exact calculation method, video frame rate interpretation, and boundary inclusion rules belong in the calibration configuration. The numbers above are **example starting values, for cold start only**.

**Failure modes:** UTC mixed with media-relative time; repeated clocks and edit offsets unhandled; wide windows stringing adjacent events together; box axis order errors; semantic similarity mistaken for the same event.

**Acceptance gate:** Time deviation, spatial mismatch, and semantic mis-association are evaluated separately; thresholds to be calibrated. Candidate relationships do not become final bindings directly.

### 6.8 Deduplication Metrics and Candidates

**Problem solved:** Quantify similar content and produce candidates for the main cleaning workflow to review. Preprocessing produces only dedup metrics and candidates, not dedup conclusions.

**General principles:**

- All raw records are kept; deduplication is not deletion.
- Same-source relationships, similarity, post counts, material counts, and account counts are saved separately.
- Cluster transitivity is not assumed: A resembling B and B resembling C does not prove A resembles C.
- Before reporting, fix three counting conventions: posts, materials, accounts; reports must state the convention used.

**Text funnel:**

1. SimHash screening: 64-bit Hamming distance `≤3` as the strict starting point; 128-bit starting point `6–8`.
2. MinHash check of candidates: long-text Jaccard `0.8` empirical value; short text under `50` characters uses `0.85–0.9` plus edit-distance or n-gram checks; number of permutations usually `128` or `256`.
3. Semantic vector check: similarity threshold to be calibrated; vector closeness is candidate evidence only.

**Image funnel:**

1. State the perceptual hash algorithm used.
2. `dHash`: fast, good for initial screening, but sensitive to cropping.
3. `pHash`: based on DCT, more robust to scaling and watermarks.
4. `aHash`: weakest of the three; it must not stand alone as the check when high discrimination is needed.
5. In the order dHash exact candidates → pHash approximate candidates → CLIP check:
   - pHash 64-bit Hamming distance `≤5` as the approximate candidate starting point.
   - CLIP cosine `>0.85` strong similarity candidate; `0.7–0.85` weak similarity, needs review; `<0.7` not counted as similar under the given convention.
   - None of these scores directly produces a dedup conclusion.

**Video candidates:**

- Compare keyframe fingerprint sequences.
- Remove intro and outro first, then compute the longest common subsequence ratio.
- Above `0.7` is a same-source candidate starting point, not a final duplication verdict.
- Shot order changes, sparse frame extraction, and speed changes can alter sequence results; record the frame extraction configuration.

**Audio candidates:**

- Audio dedup works mainly through transcription: run ASR first, then process through the text dedup funnel.
- When transcribing, keep the original audio pointer and ASR version; candidate relationships record the original audio pair, the transcript pair, and the text similarity.
- Acoustic fingerprints can serve for exact matching or initial screening, but semantic similarity or paraphrase still depends on text; method and thresholds to be calibrated.
- Audio counting conventions must state whether counting is by original audio file, by transcript, or by speaker segment.

The numbers in the conventions above are all **example starting values, for cold start only**.

**Three-convention worked example:**

- Inputs: material X posted `1` time each by `10` accounts; account A reposts `3` more times.
- Post convention: `10 + 3 = 13` posts.
- Material convention: all posts map to the same material X, counted as `1` material.
- Account convention: the accounts publishing this content are A plus `9` others, counted as `10` accounts.
- Output must state the convention chosen: by posts `13`, by materials `1`, by accounts `10`. Preprocessing only supplies this set of metrics and supporting records; whether it counts as duplication and how to handle it stays with main cleaning.

**Checklist:** Algorithm name, bit length, threshold, model version, convention, candidate chain, raw records, and cluster members are all traceable.

**Failure modes:** Same templates wrongly merged, re-encoding changing hashes, CLIP merging same-category but different materials, cluster chaining expansion, material counts mistaken for account counts.

**Acceptance gate:** Precision and recall budgets for each modality's candidates are to be calibrated; automatic record deletion or final dedup conclusions are not allowed.

---

## 7. Threshold Calibration

### 7.1 Calibration Procedure

Every threshold used for automated processing, gating, or strong evidence must trace back to labeled samples and versions. Never write an example starting value as a verified threshold.

1. **Define the task and error costs**: state the prediction target, positive/negative definitions, output action, and the impact of errors.
2. **Stratified sampling**: stratify by modality, source, clarity, length, scene, dialect, SNR, file completeness, and hard-case type. The hard-case share must be **above the natural distribution** so risks are covered effectively.
3. **Record sample versions**: sample ID, source hash, sampling method, source, time, stratum labels, annotation standard version, and evidence pointers.
4. **Two independent annotators**: annotators do not see each other's labels; disagreements go to a third person for adjudication. Keep the original labels, disagreements, and adjudicated values.
5. **Evaluate annotator agreement**: compute Cohen's kappa:
   - `>0.8`: high agreement.
   - `0.6–0.8`: moderate agreement.
   - `<0.6`: revise the annotation standard first, then re-annotate or re-review.
   - These intervals are given calibration conventions; report the actual estimate and the sample it applies to.
6. **Calibrate confidence**: bin predictions by confidence into equal-frequency bins and compare each bin's mean confidence against its actual accuracy.
7. **Compute ECE and MCE**: state bin count, boundaries, empty-bin handling, and weighting; when these implementation details are unspecified, they are to be calibrated.
8. **Choose thresholds**: on the validation set, report false positives and false negatives across threshold changes, plus the counts of automatic acceptance, human review, and rejection.
9. **Record the decision**: write the threshold, the reason for choosing it, sample version, model/rule version, false-positive / false-negative tradeoff, owner, and effective date.
10. **Redo after distribution change**: if source, content, language, resolution, noise, or event distribution changes, resample and revalidate. Never keep an old threshold as verified just because it once worked.

### 7.2 Equal-Frequency Calibration and Metric Definitions

Split prediction confidence into bins with **roughly equal sample counts**. For bin \(B_m\):

- `acc(B_m)`: empirical accuracy of predictions in the bin.
- `conf(B_m)`: mean prediction confidence in the bin.
- `|B_m| / N`: bin weight.

\[
ECE=\sum_m \frac{|B_m|}{N}\left|acc(B_m)-conf(B_m)\right|
\]

\[
MCE=\max_m\left|acc(B_m)-conf(B_m)\right|
\]

Bin count, boundaries, empty-bin policy, and ECE/MCE acceptance values are **to be calibrated**. Low ECE does not mean every subgroup is well calibrated; check by modality, source, and hard-case stratum as well, with stratum thresholds to be calibrated.

**Other calibration methods:**

Besides equal-frequency binning, choose by task:
- **Platt scaling**: for binary classification, fit a logistic regression mapping prediction scores to calibrated probabilities; needs a validation set.
- **Temperature scaling**: for multi-class classification, learn a single temperature parameter to scale logits; few parameters, generalizes well.
- **Isotonic regression**: non-parametric monotone fit; fits scores whose relationship to accuracy is monotone but nonlinear; needs more validation samples.

Method choice, validation set split, and acceptance criteria are to be calibrated for the project. This skill requires evaluating ECE/MCE and recording the calibration method used and its parameter version.

### 7.3 Error Budgets

- Set error budgets per stage. Do not let one overall average hide OCR, ASR, localization, or association errors.
- When a stage exceeds its budget: stop that stage's automatic entry into the result layer and move to review or reprocessing.
- The `1%`, `2%`, and `20%` figures in this document, when used as examples, are examples, **not standards**.
- The statistical convention, confidence intervals, and minimum sample sizes for error budgets are set by the task owner through calibration.
- Without a validated budget, do not state a "passed" quality conclusion.

### 7.4 Statistical Limits of 100 Samples

When `0` errors are observed in a sample, the 95% confidence upper bound is approximately `3 / n` by the rule of three. Therefore:

- `n = 100` with `0` errors: upper bound about `3%`.
- To verify a `1%`-level gate, at least about `300` samples are needed.
- Recommended rhythm: use `100` samples first to find workflow problems, then expand to `1000` samples for acceptance after fixes.
- This approximation serves to understand sample-size limits; whether samples are independent, how they are stratified, and how they were drawn affect the conclusion and must be stated in the report.

---

## 8. Pointers and Lineage

### 8.1 Pointer States

- `valid`: parses and resolves to the specified raw object or segment.
- `broken`: cannot be parsed, target missing, checksum mismatch, or location information insufficient to reproduce.
- `candidate_repair`: a recovery candidate exists but has not been reviewed; still counts as broken, never as valid.
- `not_applicable`: the modality or field does not apply; the reason must be stated.
- Content outside these state definitions is set by the project schema through calibration.

### 8.2 Evidence Lineage

Every standard record must trace back along this path:

```text
Result-layer record
  → this run and configuration snapshot
  → evidence-layer products with their model / rule / calibration versions
  → raw-layer files, records, and specific segments
```

Evidence-layer products record at least:

- Input `source_id` and input hash.
- Output evidence ID, target location, and pointer state.
- Model, rule, calibration, schema, and configuration versions.
- Processing time and tool / run environment identifier.
- Parameters, thresholds, and their basis.
- For corrections: raw value, corrected value, evidence grade, reason, review status, and voting information.

### 8.3 Changes and Reruns

Four regression versions must be recorded explicitly: **model, rule, schema, calibration**. Changing any one of them requires rerunning regression. When a configuration is overridden:

1. Save the default configuration version and what this run overrides.
2. Record the overridden fields, old values, new values, reasons, and applicable sources/modalities.
3. An override affects only its explicitly declared scope; it must not implicitly change other modules.
4. Form a new `config_version` from the overridden configuration.
5. When inputs, versions, or parameters change, results must support deciding whether a rerun is needed; the rules triggering a rerun are given by project configuration. Never quietly relabel old results as new-version results.

When input or processing distributions change, baselines and comparable statistics for the new distribution may be recorded; preprocessing supplies only metrics and differences and does not reach drift conclusions. Drift judgment and handling belong to the main workflow as defined by the project.

---

## 9. Conflict-Handling Boundaries and a Voting Example

Preprocessing produces conflict evidence, calibrated scores, and candidate vote calculations, but **does not adjudicate final facts**. When handing to main cleaning, the complete inputs must be kept.

### 9.1 Main-Cleaning Conflict Adjudication Order

Never reverse this order:

1. Classify: six types: numeric, unit, time, space, entity, semantic.
2. Normalize.
3. Calibrate confidence.
4. Vote.
5. Leave a trace.

Evidence grades A–E express evidence order, not probability. Evidence from the same source counts once or is down-weighted. Grade-conditional accuracy weights for voting must come from labeled-data estimates, not from the executor's subjective assignment.

### 9.2 Log-Odds Voting Worked Example

**Inputs:**

- Candidate fact `100`.
- Prior log-odds: `0`.
- ASR calibrated accuracy `0.80`, supporting `100`.
- OCR calibrated accuracy `0.60`, supporting the opposing candidate `11000`.
- Platform subtitles share a source with ASR.

**Calculation:**

- Starting log-odds: `0`.
- ASR supports `100`: add
  \(\ln(0.8/0.2)=\ln 4 \approx 1.39\).
- OCR supports `11000`, opposite in direction to `100`: subtract
  \(\ln(0.6/0.4)=\ln 1.5 \approx 0.41\).
- Platform subtitles share a source with ASR, so they are not counted again.
- Total log-odds: `0 + 1.39 − 0.41 = 0.98`.
- Posterior probability:
  \(P(100)=1/(1+e^{-0.98})\approx 0.73\).

**Output:** The posterior for `100` is about `0.73`; it does not reach the given starting gate `≥0.9`, so it goes to pending review and cannot be bound as a result.

`0.9` is an **example starting value, for cold start only**. Actual weights, calibration sets, and thresholds must be calibrated per Section 7. Preprocessing records the voting inputs, the same-source dedup basis, the formulas, and the versions; final adjudication belongs to main cleaning.

---

## 10. Preprocessing Quality Gate

The quality gate must check data, methods, and the evidence chain together. Passing does not mean conflict adjudication, deduplication, or business acceptance is done.

### 10.1 Quality Gate Report Template

```text
[Preprocessing Quality Gate Report]
source_id / data batch:
Input file hashes and inventory:
Processing time and run_id:
schema / model / rule / calibration / config versions:

A. Pointers and Coverage
Total pointers:
Valid pointers:
Broken pointers:
Integrity rate:
Result-layer record count:
No-evidence pending-review count:
Modality coverage and gaps:
Conclusion: pass / block

B. Modality Metrics
Text: encoding anomalies, CER, WER, number error rate, pending-review count
OCR: detection, CER, WER, box localization, attribution candidates, pending-review count
ASR: word-level timestamp coverage, WER, CER, number error rate, stratified results
Speakers: whether multi-speaker scenario; if applicable, DER
Video: planned / actual frame counts, trigger intervals, gradual-transition acceptance
Image/video candidates: algorithms used, parameters, candidate counts
Specific gates and sources: calibration version / to be calibrated

C. Cross-Modal Alignment
Time base and offsets:
Time tolerance and version:
Spatial coordinate system and transforms:
Semantic association method:
Pass, candidate, conflict, and pending-review counts:

D. Grey Zones and Anomalies
Grey-zone list:
Missing, damaged, broken list:
Uncovered hard cases:
Failure sample locations:

E. Baselines
Recorded input distribution:
Recorded output distribution:
Comparison scope:
Statement: recorded for later comparison only; no drift conclusion is drawn here.

F. Error Budgets
Per-stage budgets and sources:
Observed error rates and sample sizes:
Whether any budget is exceeded:
Blocking action taken when exceeded:

G. Quality Gate Conclusion
Hand over / block / needs more evidence:
Blocking reason:
Items for main cleaning to handle:
Items preprocessing did not adjudicate:
```

### 10.2 Pass Conditions

All of the following must hold before precise standard records can be handed over:

- The confirmation card is complete, and pointer integrity is `100%` before precise processing.
- Result records carry valid evidence pointers, conform to the schema, and trace back to raw values.
- Required metrics are all computed under their calibrated conventions; uncalibrated items are explicitly marked.
- No stage's error budget is exceeded; an exceeded stage stops automatic entry into the result layer.
- Candidates, conflicts, gaps, damage, and grey zones all have explicit lists.
- Versions and configuration snapshots are reproducible.
- No candidate similarity or evidence score substitutes for human / main-workflow adjudication.

When pass/fail is uncertain or metrics lack calibration, report **not yet verified / blocked**. Never write it as a pass.

---

## 11. Handover to Main Cleaning

### 11.1 Handover Interface

Main cleaning receives:

1. Unified event records and schema version.
2. Pointer mapping between result-layer records and raw-layer evidence.
3. Evidence-layer intermediate products, algorithms, parameters, and tool versions.
4. Correction audit-trail fields.
5. Grey-zone candidates, evidence, counter-evidence, and suggestions.
6. Per-modality metrics, gold standard, and error budget status.
7. Model / rule / schema / calibration / configuration version matrix.
8. Baseline distributions and their statistical conventions.
9. Blocked items, damaged files, and missing modality lists.

### 11.2 Handover Checklist Template

```text
[Handover to Main Cleaning]
source_id:
Data batch:
Preprocessing run_id:
Status: hand over / partially hand over / block
schema_version:
model_version:
rule_version:
calibration_version:
config_version:

1. Result-layer records:
   - Count:
   - Record locations:
   - Pointer integrity:

2. Evidence layer:
   - OCR:
   - ASR:
   - Video frames:
   - Cross-modal candidates:
   - Dedup metrics and candidates:
   - Other:

3. Quality gate report:
   - File / location:
   - Conclusion:
   - Failed stages:

4. Must be adjudicated by main cleaning:
   - Conflict classification and raw evidence:
   - Dedup candidates and conventions used:
   - Anomaly candidates:
   - Structured extraction tasks:
   - Review status:

5. Pending review and blocked:
   - Broken pointers:
   - Missing / damaged:
   - Threshold grey zones:
   - Insufficient evidence:
   - Uncalibrated items:

6. Changes and reruns:
   - Configuration overrides:
   - Previous version:
   - Current version:
   - Regression scope already rerun:

7. Baseline distribution record:
   - Statistics and sample scope:
   - Statement: baseline only, no drift conclusion.
```

### 11.3 Main-Cleaning Responsibility Boundary

After handover, main cleaning owns conflict adjudication, dedup conclusions, anomaly removal, structured extraction, and its own acceptance regression. Preprocessing must not write results for these steps in advance. Preprocessing is not responsible for scheduling, ingestion, version release, or report generation.

---

## 12. Testing and Regression

### 12.1 Required Test Scenarios

Tests cover normal samples, abnormal samples, and boundary samples. At minimum:

- Normal text, images, audio, video, and multimodal combinations.
- Empty values, empty files, blank text, empty audio tracks.
- Damaged, truncated, or undecodable files.
- Wrong timezones, missing timezones, media-relative time mixed with absolute time.
- Coordinate misalignment, axis order errors, missing coordinate systems, normalization out of range.
- Duplicate pointers, dead pointers, pointers to the wrong file or wrong segment.
- Encoding anomalies, garbled text, mixed encodings.
- Partial modality gaps, for example video without an audio track, or audio-only without subtitles.
- OCR box misalignment, multi-column text, subtitle changes, and gradual-transition scenes.
- Missing ASR word-level timestamps, number errors, Cantonese, Sichuan dialect, low-SNR hard cases below `10 dB`.
- Overlapping multi-speaker speech and single-speaker narration: confirm single-speaker narration does not get speaker separation.
- File names disagreeing with true formats, variable frame rate, timeline edit offsets.
- Same-source evidence double-voted because of duplicate pointers.
- Results without evidence pointers entering statistics or the result layer must be blocked.

`10 dB` is a given hard-case example, not a universal stratification boundary for all tasks; other bands still need calibration from the distribution.

### 12.2 Contents of `examples/100-sample/`

The following is a list of test assets to deliver. It does not mean the directory or files already exist:

```text
examples/100-sample/
  README.md
  manifest.jsonl
  input-inventory.csv
  hashes.txt
  annotations/
    annotator-1.jsonl
    annotator-2.jsonl
    adjudicated.jsonl
    annotation-guidelines.md
  config/
    schema.yaml
    preprocessing.yaml
    thresholds.yaml
    version-matrix.yaml
  commands/
    run.md
  metrics/
    metrics.json
    calibration.md
    error-budget.md
  failures/
    failures.jsonl
    failure-notes.md
  baselines/
    regression-baseline.json
    expected-records.jsonl
```

**Requirements for each item:**

- Fixed input inventory: sample ID, source_id, file-relative location, modality, hash, time/space metadata, hard-case stratum label, pointer state.
- Annotations: two independent results, third-person adjudication, annotation standard version, Cohen's kappa, and disagreement notes.
- Configuration: model, rule, schema, calibration, and threshold versions; uncalibrated values explicitly marked to be calibrated.
- Run commands: complete reproducible commands, input paths, configuration versions, expected output locations. Actual tool commands follow the project implementation; do not fabricate them.
- Metrics: modality splits, hard-case strata, ECE, MCE, error budgets, sample sizes, and confidence interval conventions.
- Failure samples: input hash and pointer, expected behavior, actual behavior, failure class, fix version, and whether rerun happened.
- Regression baselines: expected standard outputs and metrics under fixed inputs and configuration; allowed differences must state reasons and rules and must not silently update baselines.

### 12.3 Run and Regression Discipline

1. Fix the input inventory and version matrix so samples are not replaced by accident.
2. Run `100` samples first, aiming to find implementation, pointer, configuration, and annotation problems, without claiming `1%`-level acceptance is complete.
3. After fixes, expand to about `1000` samples for acceptance; for stricter requirements or higher stratum risk, recompute sample size from statistical requirements.
4. Rerun regression whenever any model, rule, schema, or calibration version changes.
5. Set expected behavior for normal, empty, damaged, time-space misaligned, duplicate-pointer, encoding-anomaly, and partial-modality-missing cases.
6. Check blocking logic: broken pointers must not enter precise results; error budget overruns must not auto-ingest / auto-enter the result layer.
7. Save failure samples and version information so every defect can be reproduced on fixed inputs.

---

## 13. Exclusions

This skill explicitly does not do the following:

- Adjudicate conflicts between sources, even when one source scores higher.
- Turn dedup candidates into final duplication conclusions, or delete raw records.
- Decide whether anomalies are removed.
- Treat vision / language model output as verified structured facts.
- Backfill inferred missing times, coordinates, speakers, or objects as observed values.
- Draw drift conclusions from baseline differences.
- Own scheduling, data ingestion, version release, or operations reports.
- Claim targets are met for uncalibrated thresholds.
- Treat evidence grades A–E as probabilities.
- Let results without pointers take part in precise statistics.

---

## 14. Common Pitfalls

1. **Extract first, add pointers later**: conclusions become unverifiable. Pointers come first; damage blocks.
2. **Raw values overwritten by standardized values**: unrecoverable. Raw values are always kept; standardized values are derived fields.
3. **Reporting a single overall accuracy**: hides failures in numbers, low-SNR dialects, gradual-transition scenes, and spatial attribution. Report split by modality and hard-case stratum.
4. **Treating strong IoU as binding**: box overlap is only evidence; time overlap and context confirmation are still needed.
5. **Using sentence-level ASR timestamps for word-level alignment**: does not meet word-level localization requirements.
6. **Forcing speaker separation on single-speaker narration**: violates this skill's applicability rules; speaker separation applies to multi-speaker scenarios only.
7. **Processing all video frame by frame, or sampling at a fixed sparse rate**: the first costs too much, the second can miss events. Use the pre-defined trigger-based base-plus-densification strategy and verify gradual-transition cuts.
8. **Using a `500 ms` tolerance for all events**: it may be too wide for fast-paced narration and can cross adjacent events. Calibrate by frame rate, scene, and event interval.
9. **Treating calibrated scores as true probabilities**: without labeled data, equal-frequency binning, ECE/MCE, and subgroup checks, the scores are unverified.
10. **Double-voting same-source subtitles and ASR**: inflates evidence. Identify source correlation; same-source evidence counts once or is down-weighted.
11. **Mapping evidence grades to probabilities**: A–E are ordinal grades only.
12. **Treating approximate similarity as dedup**: hashes, embeddings, and LCS are metrics or candidates; final adjudication sits with main cleaning.
13. **Assuming cluster transitivity**: A–B and B–C similar does not prove A–C identical.
14. **Reporting statistics without stating the convention**: post, material, and account results can differ completely. Fix the convention before reporting.
15. **Thresholds without versions and effective dates**: impossible to reproduce later. Sample version, false-positive / false-negative tradeoff, and effective date must be recorded together.
16. **Claiming 1% from 100 samples with zero errors**: with zero errors, the 95% upper bound is still about `3%`; increase sample size for the target risk.
17. **Writing "not seen" as "does not exist"**: frame extraction, OCR, and ASR can all miss. Express it as insufficient coverage or insufficient evidence.
18. **Auto-repairing broken pointers**: a candidate recovery is not a repair; keep it broken and re-verify pointers after pending review completes.
19. **Overriding default configuration without a trace**: breaks reproducibility. Override values and reasons go into the configuration version.
20. **Writing to be calibrated as a default gate**: a number without basis is not a standard.

---

## 15. Short Verification Checklist

**Adopted given conventions:**

- Confirmation card fields; pointer integrity `100%` required before precise processing; `broken` blocks the result layer.
- Three-layer storage, unified events, correction audit trails, and version fields.
- OCR detection and recognition kept separate; CER/WER reported separately; IoU, containment ratio, center distance, and time-space context attribution rules.
- ASR word-level timestamps, WER/CER, number error rate, multi-speaker vs. single-speaker handling rules, and hard-case stratification.
- Trigger-based densified video frame extraction, single densification cap, cost formula, the `124`-frame worked example, and dedicated gradual-transition acceptance.
- Dynamic time tolerance, per-scenario examples, and the adjacent-event interval constraint.
- Conflict adjudication order, equal-frequency calibration, ECE/MCE, same-source down-weighting, and the log-odds voting example.
- Text, image, video, and audio dedup candidate funnels, three counting conventions, and the `13/1/10` worked example.
- Gold standard, error budgets, four-version matrix, and sample size notes.
- End-to-end processing, handover templates, quality gate, testing, and the `examples/100-sample/` contents list.

**Still to be calibrated or determined by the project:**

- File format and hash implementation details, run_id generation rule, schema field serialization form.
- OCR/ASR/detection/localization/alignment acceptance gates, center distance threshold, bin counts, and confidence calibration acceptance values.
- Per-dataset stratum boundaries, sampling ratios, dialect and remaining SNR bands.
- Video base rate, densified rate, trigger scope policy, and frame extraction coverage gates.
- Time tolerance configuration and time base implementation.
- MinHash / semantic model configuration, video fingerprint implementation, and per-candidate error budgets.
- Error budgets and corresponding statistical conventions for each module.
- Calibration method choice (Platt / temperature / isotonic regression), ECE/MCE gates, false-positive / false-negative tradeoff, effective dates, and conditions for recalibration after distribution change.
- Actual project run commands, input file names, and fixed sample contents.

The items above that have not been given must be filled in through annotation and validation in the actual project. They cannot be replaced by example starting values.
