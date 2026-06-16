# symbolic_music_features Feature Design

## 1. Document Purpose

This document explains the current feature extraction design of the
`symbolic_music_features` project in implementation-level detail.

It focuses on:

- the normalized event table that all downstream logic depends on
- shared helper functions and their assumptions
- the implementation logic of each feature family
- the distinction between diagnostic features and modeling features
- the normalization rules used before PCA, GMM, clustering, or other modeling
- the proxy nature and analytical limits of the current design

This is a design explanation of the current codebase, not a wish-list for future
work. When behavior is approximate, heuristic, or intentionally simplified, it
is described explicitly.

## 2. Pipeline Overview

The end-to-end pipeline is:

`MusicXML / MXL / XML -> music21 Score -> normalized event table -> fixed-bar segments -> segment-level feature extraction -> diagnostic/modeling split -> optional work-level aggregation`

The key implementation choices are:

- all feature extractors work from a normalized pandas event table, not directly
  from `music21` objects
- segmentation is bar-based and happens before feature extraction
- each feature family emits a stable set of columns
- raw count-like features are preserved for diagnostics but are excluded from the
  default modeling matrix
- modeling features are family-aware normalized to prevent large families from
  dominating PCA or clustering

## 3. Normalized Event Table

### 3.1 Core idea

The event table is produced by `score_to_event_table()` in
`src/symbolic_music_features/representation/score_to_events.py`.

Each row is an event-like symbolic unit. Notes and rests are represented
directly. Chords are split into one row per pitch while retaining
`is_chord=True`.

This design is central to the whole project:

- pitch-oriented families can use pitch-split rows directly
- rhythm-oriented families can collapse those rows back with
  `deduplicate_rhythm_events()`
- line-based interval and motif logic can group rows by encoded voice
- onset-based vertical features can compare all pitches active at the same onset

### 3.2 Important columns

The most important fields used across the feature families are:

- `work_id`, `movement_id`, `composer`, `source_path`
- `part_id`, `part_name`, `part_index`
- `voice_id`
- `measure_no`
- `measure_physical_index`
- `global_onset_ql`
- `measure_onset_ql`
- `duration_ql`
- `offset_ql`
- `pitch_midi`
- `pitch_name`
- `pitch_class`
- `is_rest`
- `is_chord`
- `chord_size`
- `tie_type`
- `is_tied`
- `beat`
- `beat_strength`
- `time_signature`
- `key_signature`
- `key_signature_fifths`
- `key_mode`

### 3.3 Why `measure_physical_index` matters

`measure_no` is the printed measure number, which may repeat, skip, or contain
editorial numbering issues. `measure_physical_index` is a zero-based physical
order index assigned during traversal.

The project prefers `measure_physical_index` for:

- rhythm-event deduplication
- segment boundary logic
- stable within-score ordering

### 3.4 Voice handling

The event conversion explicitly traverses `music21.stream.Voice` containers when
present. If no voice objects are present, a default voice id is synthesized per
part.

This is important because many later features use `(part_id, voice_id)` as an
encoded stream key.

Important limit:

- `voice_id` is an encoded notation-layer stream id
- it is not guaranteed to be a recovered analytical voice
- all voice-based texture, interval, and motif features must therefore be read as
  encoded-voice proxies

### 3.5 Chord splitting

Chord rows are split by pitch. This gives better support for:

- pitch histograms
- vertical interval classes
- sonority-level texture summaries

But it also creates a risk:

- if we count rows naively, one chord can be mistaken for several rhythmic
  events or several active voices

To compensate, several helpers intentionally deduplicate by part, voice, onset,
and duration when the task is rhythmic or onset-structural rather than
pitch-structural.

## 4. Shared Helper Design

Most feature families are built from a small shared helper layer in
`src/symbolic_music_features/utils/music.py`.

### 4.1 `get_note_events()`

Purpose:

- filter the event table down to non-rest rows with concrete pitch information

Behavior:

- retains chord-split rows
- removes rest rows
- returns an empty frame if required columns are absent

Used by:

- pitch
- interval
- tonal
- texture/polyphony
- motif

### 4.2 `get_segment_duration()`

Purpose:

- estimate segment duration as `max(offset_ql) - min(global_onset_ql)`

Why it exists:

- note density and related features need a stable denominator

Limit:

- this is an event-span estimate, not a meter-aware notated duration model

### 4.3 `deduplicate_rhythm_events()`

Purpose:

- collapse chord-split rows back to onset-duration events

Deduplication keys prioritize:

- `part_index`
- `part_id`
- `voice_id`
- `measure_physical_index`
- `measure_onset_ql`
- `global_onset_ql`
- `duration_ql`
- `offset_ql`
- `is_rest`
- `is_chord`

Why it matters:

- a chord should usually count as one rhythmic event inside a voice, not several
  pitch events

Used by:

- rhythm family
- motif rhythm-token logic

### 4.4 `group_melodic_lines()`

Purpose:

- group events by `(part_id, voice_id)` and reduce simultaneous pitches at the
  same onset to one proxy pitch

Supported proxies:

- `"highest"`
- `"lowest"`
- `"mean"`

Why it matters:

- interval and motif features need line-like sequences
- keyboard textures often contain chordal simultaneities inside one encoded voice

Important limit:

- this is not voice separation
- it is a reduction of simultaneous pitches inside an encoded stream

### 4.5 `collect_encoded_voice_linear_intervals()`

Purpose:

- compute adjacent pitch differences inside encoded part/voice streams

Interpretation:

- encoded-voice motion proxy
- not a full melodic-line reconstruction

### 4.6 `reduce_to_global_onset_line()` and `collect_onset_contour_intervals()`

Purpose:

- derive onset-based top-line or bass-line contour proxies

Method:

- at each onset, reduce all active pitches to either the highest pitch or the
  lowest pitch
- then compute adjacent onset-to-onset pitch intervals

Interpretation:

- onset contour proxy
- useful for comparing upper contour or bass contour motion
- not a complete voice-leading extraction

### 4.7 `collect_vertical_interval_classes()` and `pairwise_interval_classes()`

Purpose:

- summarize vertical pitch relations present at each onset

Method:

- group note rows by `global_onset_ql`
- compute pairwise interval classes between simultaneous pitches

Important limit:

- this is onset-based vertical structure
- sustained durations across later onsets are intentionally ignored
- it is not a full duration-aware sounding-sonority model

### 4.8 `windowed_pitch_class_histograms()`

Purpose:

- build local pitch-class histograms on a fixed onset-based time grid

Current default:

- `window_size_ql = 4.0`

Used by:

- tonal local diversity proxy

### 4.9 `safe_ratio()`

Purpose:

- provide stable ratio computation

Behavior:

- returns `NaN` if denominator is zero or missing

This is a design principle used repeatedly to avoid silent divide-by-zero
artifacts.

## 5. Segment-Level Feature Architecture

All segment-level families inherit from `FeatureExtractor` in
`src/symbolic_music_features/features/base.py`.

Each extractor defines:

- `family`
- `feature_names`
- `extract_segment()`

Shared guarantees:

- every extractor returns a stable set of output columns
- failures fall back to `empty_result()`
- `safe_extract_segment()` preserves pipeline robustness

The feature registry in `features/registry.py` instantiates the default family
set:

- pitch
- interval
- rhythm
- tonal
- polyphony
- motif_sequence
- geometry

## 6. Pitch Family

Implementation file:

- `src/symbolic_music_features/features/pitch.py`

### 6.1 Design goal

The pitch family captures:

- pitch register
- pitch spread
- pitch-class usage
- the relative location of the local climax

It is intentionally distributional and descriptive rather than harmonic.

### 6.2 Input strategy

The extractor uses `get_note_events()`, which means:

- rests are excluded
- chord-split rows are retained

This makes the pitch family event-dense by design. Chords contribute multiple
pitched rows.

### 6.3 Feature groups

#### A. Raw note-event count

- `pitch_count`

Meaning:

- number of pitched note rows in the segment

Status:

- diagnostic by default
- excluded from modeling unless raw counts are enabled

#### B. Register and spread

- `pitch_range`
- `pitch_mean`
- `pitch_std`
- `pitch_min`
- `pitch_max`

Implementation:

- computed directly from `pitch_midi`

Interpretation:

- these are straightforward register summaries
- because chord pitches are retained, dense chords can influence the mean and
  variance strongly

#### C. Pitch-class distribution

- `pitch_class_entropy`
- `pitch_class_0` ... `pitch_class_11`

Implementation:

- build a 12-bin pitch-class histogram from `pitch_class`
- normalize to proportions
- compute Shannon entropy over the unnormalized histogram

Interpretation:

- entropy measures pitch-class dispersion
- the twelve `pitch_class_*` features form a normalized pitch-class profile

#### D. Climax location

- `climax_relative_position`

Implementation:

- find the highest pitch in the segment
- among all rows with that pitch, take the earliest onset
- compute `(earliest_climax - segment_start) / segment_duration`

Interpretation:

- relative timing of the first local apex

Current empty-segment behavior:

- the extractor sets `pitch_count = 0`
- all pitch-class proportions are set to `0.0`
- other fields remain `NaN`

This is slightly older behavior than the newer strict-`NaN` proxy families and
is worth remembering when building modeling workflows.

## 7. Interval Family

Implementation file:

- `src/symbolic_music_features/features/interval.py`

### 7.1 Design goal

The interval family separates linear and vertical motion into explicit proxy
sources so that later analysis can distinguish:

- encoded voice motion
- upper contour motion
- bass contour motion
- onset-based vertical sonority relations

### 7.2 Linear interval sources

There are three current linear sources:

- `encoded_voice`
- `top_line`
- `bass_line`

The implementation computes each source separately rather than collapsing them
into one generic interval pool.

### 7.3 Legacy diagnostic fields

- `interval_count`
- `interval_mean_abs`
- `interval_std`
- `interval_entropy`

These are built from encoded-voice intervals only.

Design role:

- backward compatibility
- diagnostics

Modeling status:

- excluded from the default modeling matrix

### 7.4 Linear interval summary features

For each source, the extractor emits:

- `repeat_ratio`
- `step_ratio`
- `small_leap_ratio`
- `medium_leap_ratio`
- `large_leap_ratio`
- `ascending_ratio`
- `descending_ratio`
- `direction_change_rate`
- `interval_entropy`
- `mean_abs_interval`

The full feature names are:

- `interval_encoded_voice_*`
- `interval_top_line_*`
- `interval_bass_line_*`

Implementation details:

- `repeat_ratio`: proportion of intervals equal to `0`
- `step_ratio`: absolute interval in `{1, 2}`
- `small_leap_ratio`: absolute interval in `{3, 4}`
- `medium_leap_ratio`: absolute interval in `{5, 6}`
- `large_leap_ratio`: absolute interval `>= 7`
- `ascending_ratio`: interval `> 0`
- `descending_ratio`: interval `< 0`
- `direction_change_rate`: sign change rate over nonzero intervals only
- `interval_entropy`: entropy of clipped interval values in `[-12, 12]`
- `mean_abs_interval`: mean absolute interval size

Why clipping is used:

- interval entropy and histograms should not be dominated by a few extreme
  outliers

### 7.5 Signed histogram features

Currently emitted for:

- `top_line`
- `bass_line`

Optionally also for:

- `encoded_voice`

Bins:

- `-12` through `12`

Feature names:

- `interval_top_line_hist_-12` ... `interval_top_line_hist_12`
- `interval_bass_line_hist_-12` ... `interval_bass_line_hist_12`

Interpretation:

- normalized directional interval profiles

### 7.6 Vertical interval features

The vertical portion is onset-based.

Feature groups:

#### A. Per-class interval ratios

- `vertical_ic_0_ratio` ... `vertical_ic_11_ratio`

#### B. Compact category ratios

- `vertical_unison_octave_ratio`
- `vertical_second_seventh_ratio`
- `vertical_third_sixth_ratio`
- `vertical_fourth_fifth_ratio`
- `vertical_tritone_ratio`

These are grouped from interval classes as follows:

- unison/octave: `0`
- second/seventh: `1, 2, 10, 11`
- third/sixth: `3, 4, 8, 9`
- fourth/fifth: `5, 7`
- tritone: `6`

#### C. Density and entropy

- `vertical_interval_count`
- `vertical_interval_count_per_onset`
- `vertical_interval_entropy`

Interpretation:

- `vertical_interval_count` is raw and diagnostic
- `vertical_interval_count_per_onset` is normalized and modeling-eligible
- entropy measures the diversity of vertical interval classes

### 7.7 Limits

The interval family is musically richer than a generic pooled-interval design,
but it still has strong limits:

- encoded voices are notation-level proxies
- top and bass lines are onset reductions, not recovered voices
- vertical features ignore sustained duration beyond the onset

## 8. Rhythm Family

Implementation file:

- `src/symbolic_music_features/features/rhythm.py`

### 8.1 Design goal

The rhythm family captures:

- event density
- duration distribution
- rest proportion
- dotted-rhythm tendency
- a simplified syncopation signal

### 8.2 Input strategy

The extractor starts with `deduplicate_rhythm_events()`.

This is crucial because:

- chords should count as one rhythmic event inside a voice
- rhythm should not inherit pitch-row multiplicity from chord splitting

### 8.3 Feature groups

#### A. Event count and density

- `rhythm_event_count`
- `rhythm_note_density_per_ql`

Implementation:

- `rhythm_event_count`: number of deduplicated rhythm rows
- `rhythm_note_density_per_ql`: non-rest event count divided by event-span duration

Status:

- `rhythm_event_count` is diagnostic by default
- density is modeling-oriented

#### B. Rest and duration summaries

- `rhythm_rest_ratio`
- `rhythm_duration_mean`
- `rhythm_duration_std`
- `rhythm_duration_entropy`

Interpretation:

- rest ratio summarizes silence proportion at the event level
- duration entropy measures rhythmic variety across deduplicated durations

#### C. Dotted rhythm tendency

- `rhythm_dotted_ratio`

Implementation:

- uses `duration_is_dotted()`
- matches against a fixed list of common dotted quarter-length values with a
  tolerance

Interpretation:

- lightweight dottedness heuristic
- not a full rhythmic notation parser

#### D. Beat-strength summaries

- `rhythm_beat_strength_mean`
- `rhythm_syncopation_proxy`

Current syncopation proxy:

- find events with `beat_strength < 0.5`
- among those, compute the proportion with `duration_ql >= 1.0`

Interpretation:

- longish events starting on weak beats are treated as syncopation-like

Limit:

- this is intentionally simple
- it does not model metrical dissonance, tied syncopation chains, or phrase-level
  metric displacement

### 8.4 Empty behavior

If the deduplicated rhythmic frame is empty:

- `rhythm_event_count = 0`
- most other values remain `NaN`

As with pitch, this reflects an older family design that mixes explicit zero
counts with `NaN` summaries.

## 9. Tonal Family

Implementation file:

- `src/symbolic_music_features/features/tonal.py`

### 9.1 Design goal

The tonal family avoids full harmonic reduction and instead provides stable,
lightweight tonal proxies based on:

- key signature context
- local pitch-class distributions
- chromatic pitch-class incidence
- local distributional change across windows

### 9.2 Key context strategy

The event table stores:

- `key_signature`
- `key_signature_fifths`
- `key_mode`

These are resolved measure by measure during score conversion.

Important design choice:

- explicit `KeySignature` and explicit analytical `Key` are tracked separately
- quantitative features prefer `key_signature_fifths` when possible because it is
  usually more stable than analytical key labels

### 9.3 Feature groups

#### A. Key signature and mode summaries

- `tonal_key_signature_fifths_mean`
- `tonal_major_key_ratio`
- `tonal_minor_key_ratio`

Implementation:

- mean over available `key_signature_fifths`
- ratio of rows whose `key_mode` equals `"major"` or `"minor"`

Interpretation:

- segment-level tonal context summary

#### B. Pitch-class entropy

- `tonal_pitch_class_entropy`

Implementation:

- note-only pitch-class histogram

Interpretation:

- how concentrated or diffuse the pitch-class usage is

#### C. Chromaticism proxy

- `tonal_chromaticism_ratio`

Implementation:

- for each note row, use `diatonic_pitch_classes()` to approximate the diatonic
  set implied by `key_signature_fifths` and `key_mode`
- count notes whose pitch class falls outside that set

Interpretation:

- a simple in-key vs out-of-key pitch-class proxy

Important limit:

- minor-mode variants are not modeled fully
- raised leading tones and melodic-minor behavior are not explicitly encoded

#### D. Local key diversity proxy

- `tonal_local_key_diversity_proxy`

Implementation:

- build onset-based pitch-class histograms in fixed `4.0 ql` windows
- normalize each histogram
- compute Jensen-Shannon distance between adjacent windows
- take the mean

Interpretation:

- local tonal distributional change
- higher values suggest greater local reconfiguration

Limit:

- this is not literal modulation detection
- it is a local distributional divergence measure

## 10. Texture / Polyphony Family

Implementation file:

- `src/symbolic_music_features/features/polyphony.py`

### 10.1 Naming note

The extractor class is still called `PolyphonyFeatureExtractor`, and it still
emits legacy `polyphony_*` columns. Newer texture-oriented features are emitted
with the `texture_*` prefix.

In normalization, both `polyphony_*` and `texture_*` are mapped to the same
logical `texture` family.

### 10.2 Design goal

This family now combines two layers:

- legacy simultaneity and sonority summaries
- newer encoded active voice, stability, synchrony, and motion proxies

The design intentionally avoids claiming full counterpoint analysis.

### 10.3 Legacy polyphony features

#### A. Active voice and simultaneity size

- `polyphony_active_voice_mean`
- `polyphony_active_voice_max`
- `polyphony_simultaneity_mean`
- `polyphony_simultaneity_max`

Implementation:

- grouped by onset
- active voice count uses distinct `(part_id, voice_id)` pairs
- simultaneity count uses the number of pitch rows at the onset

Interpretation:

- `active_voice_*` approximates texture thickness by encoded streams
- `simultaneity_*` reflects raw pitch multiplicity

#### B. Chord event ratio

- `polyphony_chord_event_ratio`

Implementation:

- mean of `is_chord` over note rows

Interpretation:

- how often note events belong to chord objects

#### C. Vertical entropy and rough consonance balance

- `polyphony_vertical_interval_entropy`
- `polyphony_consonance_ratio`
- `polyphony_dissonance_ratio`

Implementation:

- compute pairwise onset interval classes
- consonant set is currently `{0, 3, 4, 5, 7, 8, 9}`

Interpretation:

- coarse sonority-balance proxy

Limit:

- this is not species counterpoint or tonal dissonance treatment analysis

### 10.4 Active voice distribution

New feature group:

- `texture_active_voice_1_ratio`
- `texture_active_voice_2_ratio`
- `texture_active_voice_3_ratio`
- `texture_active_voice_4_ratio`
- `texture_active_voice_5plus_ratio`
- `texture_active_voice_mean`
- `texture_active_voice_std`
- `texture_active_voice_entropy`

Implementation:

- first deduplicate chord-split rows by `global_onset_ql` and encoded voice key
- count active encoded voices at each onset
- summarize the distribution across the segment

Why deduplication matters:

- one chord inside one encoded voice must not be mistaken for several active
  voices

Interpretation:

- onset-based encoded active voice profile

Limit:

- encoded active voice count is not equivalent to analytical part count

### 10.5 Texture stability and change rate

Feature group:

- `texture_voice_count_change_rate`
- `texture_voice_count_change_mean_abs`
- `texture_voice_count_change_max_abs`
- `texture_voice_count_change_std_abs`
- `texture_voice_count_stability_ratio`

Implementation:

- build the ordered onset sequence of active encoded voice counts
- compute absolute adjacent differences
- summarize whether adjacent onsets change and by how much

Interpretation:

- whether texture thickness is stable or fluctuating

This is one of the cleanest examples of the project philosophy:

- musically meaningful
- cheap to compute
- clearly labeled as an encoded-voice texture proxy

### 10.6 Rhythmic independence proxy

Feature group:

- `texture_homorhythmic_onset_ratio`
- `texture_staggered_onset_ratio`

Implementation:

- reuse the active-voice-count-on-onset sequence
- onset count `>= 2` is treated as multi-voice simultaneous attack
- onset count `== 1` is treated as single-voice attack

Interpretation:

- onset synchrony proxy

Limit:

- this does not distinguish homophony from all other textures in a rigorous way
- it is only a note-onset coordination measure

### 10.7 Directional independence and motion-type proxies

Feature group:

- `texture_direction_agreement_ratio`
- `texture_direction_disagreement_ratio`
- `texture_direction_mixed_ratio`
- `texture_parallel_motion_ratio`
- `texture_contrary_motion_ratio`
- `texture_oblique_motion_ratio`

Implementation steps:

1. group notes by encoded voice
2. reduce same-onset multiple pitches inside a voice to their mean pitch
3. compute adjacent interval direction per voice
4. attach each direction to the interval's starting onset
5. compare all voice-direction pairs that occur on the same onset

Definitions:

- agreement: same nonzero direction
- disagreement: opposite signs
- mixed: everything else
- parallel: same nonzero direction
- contrary: opposite nonzero direction
- oblique: one voice moves, the other repeats

Interpretation:

- lightweight voice independence proxy

Limit:

- the design uses direction only
- it does not model interval-size-aware similar motion separately
- it does not reconstruct voice-leading across entry/exit ambiguities

## 11. Motif / Sequence Family

Implementation file:

- `src/symbolic_music_features/features/motif_sequence.py`

### 11.1 Design goal

The motif family was redesigned to reduce dependence on raw unique-count
features. The current design emphasizes:

- recurrent local symbolic patterns
- candidate sequence-like patterns
- lightweight self-similarity

It explicitly avoids claiming:

- automatic motif detection
- thematic parsing
- true sequence analysis in the strict musicological sense

### 11.2 Tokenization strategy

This family is token-based.

There are three main token streams:

#### A. Interval tokens

Source:

- encoded voice lines built by `group_melodic_lines(..., melodic_proxy="mean")`

Token form:

- `i<interval>`
- for example `i2`, `i-3`, `i0`

Why `mean` is used:

- same-onset multiple pitches inside one encoded voice are reduced to a neutral
  proxy pitch
- this avoids always privileging upper-note reduction

#### B. Rhythm tokens

Source:

- note-only rhythmic events after `deduplicate_rhythm_events()`

Token form:

- `dur_<duration>`
- rounded to three decimal places

#### C. Interval-rhythm cell tokens

Source:

- interval token paired with the starting note duration in the same encoded line

Token form:

- `i<interval>_dur_<duration>`

This is the default sequence used for self-similarity because it combines motion
and duration.

### 11.3 Recurrent local pattern features

Feature group:

- `motif_repeated_interval_bigram_ratio`
- `motif_repeated_interval_trigram_ratio`
- `motif_repeated_rhythm_bigram_ratio`
- `motif_repeated_interval_rhythm_cell_ratio`

Implementation:

- build n-grams from token streams
- count how many total n-gram occurrences belong to types appearing at least
  twice

Current formula:

`repeated_ngram_ratio = repeated_occurrences / total_ngram_occurrences`

Interpretation:

- normalized local repetition strength

Why this is better than raw unique count:

- it measures how much of the sequence is structurally repetitive
- it is less directly dominated by segment length

### 11.4 Exact repetition proxy

Feature:

- `motif_exact_repetition_ratio`

Current implementation:

- build pitch trigrams from encoded voice pitch tokens
- compute repeated n-gram ratio over those pitch windows

Interpretation:

- literal local pitch-sequence recurrence

Limit:

- the current implementation uses fixed trigram windows
- it is a local exact repetition proxy, not a generic long-range repeated theme

### 11.5 Transposed repetition proxy

Feature:

- `motif_transposed_repetition_ratio`

Implementation:

1. build pitch windows of length `3` from encoded voice pitch lines
2. convert each pitch window into its interval pattern
3. group windows by interval pattern
4. count windows where the same interval pattern occurs with different starting
   pitches

Interpretation:

- candidate transposed local sequence

This is one of the most directly music-relevant motif features in the current
codebase because it explicitly ignores absolute pitch level while preserving
interval pattern.

### 11.6 Direction-consistent interval pattern repetition

Feature:

- `motif_directional_pattern_repetition_ratio`

Implementation:

- convert intervals into coarse direction/size tokens:
  - `R`
  - `U_step`
  - `D_step`
  - `U_small_leap`
  - `D_small_leap`
  - `U_large_leap`
  - `D_large_leap`
- then compute repeated bigram ratio over those tokens

Interpretation:

- repeated motion-shape tendencies even when the exact intervals differ

### 11.7 Sequence-like continuation features

Feature group:

- `motif_sequence_like_ratio`
- `motif_sequence_run_mean_length`
- `motif_sequence_run_max_length`

Implementation:

- build adjacent interval windows of length `2`
- also build adjacent direction-class windows of length `2`
- treat a continuation as true when adjacent windows share either:
  - the same exact interval pattern
  - or the same direction-class pattern

Interpretation:

- how often local windows continue a repeated or shape-consistent pattern
- how long such runs tend to last

Limit:

- this is an intentionally permissive local continuation heuristic
- it is not equivalent to formal sequential writing detection

### 11.8 Self-similarity features

Feature group:

- `motif_self_similarity_mean`
- `motif_self_similarity_max`
- `motif_self_similarity_entropy`

Implementation:

- default token stream is interval-rhythm cell tokens
- build sliding windows of length `4`
- compare every pair of windows using Jaccard similarity over token sets
- summarize the off-diagonal similarities

Definitions:

- mean: average similarity
- max: strongest similarity
- entropy: histogram entropy of similarity values using 10 bins from `0.0` to
  `1.0`

Interpretation:

- lightweight internal repetition landscape

Limit:

- order inside a window is reduced to set overlap for the similarity measure
- this is intentionally cheaper and simpler than alignment or DTW

### 11.9 Backward-compatible diagnostic fields

Still emitted:

- `motif_interval_bigram_unique_count`
- `motif_interval_bigram_unique`
- `motif_interval_trigram_unique`
- `motif_interval_bigram_repetition_rate`
- `motif_interval_trigram_repetition_rate`
- `motif_rhythm_bigram_unique`
- `motif_rhythm_trigram_unique`
- `motif_compression_ratio`

Current role:

- diagnostics
- backward compatibility with earlier experiments

Modeling status:

- unique/count-like fields are excluded from the default modeling matrix

## 12. Geometry Family

Implementation file:

- `src/symbolic_music_features/features/geometry.py`

### 12.1 Design goal

The geometry family captures repeated local interval shapes using a more
token-document style representation. It is designed mainly for later
unsupervised workflows.

### 12.2 Token construction

`build_interval_token_string()`:

- groups melodic lines by encoded voice
- converts adjacent pitch differences into interval tokens
- if enough intervals exist, concatenates short interval windows with `"|"`

Example idea:

- single intervals if the line is short
- short interval shingles when enough material exists

### 12.3 Segment-level geometry features

Feature group:

- `geometry_self_similarity_mean`
- `geometry_self_similarity_std`
- `geometry_self_similarity_max`
- `geometry_pattern_entropy`
- `geometry_interval_tfidf_ready_token_count`

Implementation:

- split the token string into tokens
- compute token-count entropy
- build one-hot vectors for each token occurrence
- compute pairwise cosine similarity between those token vectors
- summarize the upper triangle

Interpretation:

- how repetitive the token geometry is
- how concentrated token usage is

Important observation:

- because the vectors are one-hot, cosine similarity is effectively equality of
  token identity
- this family therefore emphasizes recurrence of identical interval-shape tokens
  rather than graded geometric similarity

### 12.4 TF-IDF export helper

`build_tfidf_features()` is not a per-segment extractor. It is a downstream
utility that turns precomputed token strings into a segment-by-token TF-IDF
matrix.

This is intended for:

- corpus-level similarity modeling
- document-style unsupervised exploration

## 13. Diagnostic vs Modeling Features

Implementation file:

- `src/symbolic_music_features/modeling/normalization.py`

### 13.1 Why the split exists

Raw extracted features are useful for:

- debugging
- plotting
- sanity checking
- analytical inspection

But the same raw table is not ideal for PCA or clustering because:

- count features grow with segment size and event density
- high-dimensional families can dominate distance geometry
- heavy-tailed features destabilize unsupervised models

The project therefore writes two segment-level matrices:

- `diagnostic_features`
- `modeling_features`

### 13.2 Family inference

Family membership for normalization is prefix-based.

Current mapping:

- `pitch_`, `climax_` -> `pitch`
- `interval_`, `vertical_` -> `interval`
- `rhythm_` -> `rhythm`
- `tonal_` -> `tonal`
- `polyphony_`, `texture_` -> `texture`
- `motif_` -> `motif`
- `geometry_` -> `geometry`

This mapping is deliberately independent from extractor class names.

### 13.3 Feature kinds

The normalizer classifies numeric features into:

- `ratio`
- `density`
- `count`
- `continuous`

Current rules:

- ratio:
  - column contains `_hist_`
  - or starts with `pitch_class_`
  - or starts with `vertical_ic_`
  - or ends with `_ratio`
- density:
  - contains `_per_bar`, `_per_ql`, `_per_event`, `_per_onset`, or `_density_`
- count:
  - ends with `_count`, `_raw`, `_raw_count`, `_unique`, or `_unique_count`
- continuous:
  - everything else numeric

### 13.4 Default modeling inclusion

By default:

- ratio features are included
- density features are included
- count-like features are excluded
- continuous features are included

Important special case:

- legacy interval diagnostics
  - `interval_count`
  - `interval_mean_abs`
  - `interval_std`
  - `interval_entropy`
  are explicitly excluded

### 13.5 Transformation rules

Default transformations:

- ratio features:
  - kept on original scale
- density features:
  - `log1p` if nonnegative
  - then z-score
- count features:
  - `log1p` if nonnegative
  - then z-score
  - but excluded unless `use_raw_counts=True`
- continuous features:
  - z-score

### 13.6 Family-level L2 normalization

After per-column transforms, the normalizer optionally applies L2 normalization
within each family per row.

Goal:

- stop a family with many columns from overpowering a smaller family

Example:

- interval histograms can easily dominate a low-dimensional tonal family if the
  combined vector is left unbalanced

### 13.7 NaN handling

The normalizer:

- preserves NaN positions after family normalization
- uses `np.nan_to_num(..., nan=0.0)` only internally for norm calculation

This is an important design choice because:

- missingness should not be silently converted into real signal

## 14. Output Files and Their Roles

The main extraction pipeline writes:

- `events`
- `segments`
- `segment_features`
- `diagnostic_features`
- `modeling_features`
- `work_features`
- `feature_registry.csv`
- `feature_normalization_report.csv`

Meaning:

- `segment_features`: raw direct extractor output before modeling split
- `diagnostic_features`: raw features with identifiers, kept for inspection
- `modeling_features`: transformed modeling-ready matrix with identifiers
- `work_features`: aggregated work-level summaries from segment features
- `feature_registry.csv`: feature metadata
- `feature_normalization_report.csv`: per-feature normalization decisions and
  before/after statistics

## 15. Current Design Principles

Several design principles appear repeatedly across the project.

### 15.1 Proxy over false precision

The system prefers clearly labeled lightweight proxies over brittle
high-pretension analysis.

Examples:

- encoded voice motion instead of claimed voice reconstruction
- onset contour instead of fully separated melody extraction
- transposed repetition proxy instead of claimed motif detection
- weak-beat duration heuristic instead of formal syncopation theory

### 15.2 Stable denominators

Ratio features use explicit, stable denominators whenever possible:

- count of intervals
- count of onsets
- count of windows
- segment event-span duration

This keeps features more comparable across segments.

### 15.3 Diagnostic preservation

The project does not discard raw count-like features entirely.

Instead, it keeps them for:

- debugging
- corpus sanity checks
- interpretability

But it avoids letting them dominate unsupervised geometry by default.

### 15.4 Segment-first design

All core features are segment-level first.

This matches the intended downstream usage:

- PCA
- GMM
- KMeans
- work-level aggregation over segment distributions

## 16. Important Analytical Limits

The current system is practical and musically informed, but it is not a full
computational musicology engine. The main limits are:

- encoded `voice_id` is not guaranteed to equal analytical voice identity
- onset-based vertical features ignore sustained sonorities across later onsets
- syncopation is heuristic
- tonal features rely heavily on key-signature context and pitch-class behavior
- motif features capture local recurrence, not formal themes
- geometry features mostly reward repeated token identity
- different families still have slightly different empty-segment conventions

In particular:

- newer texture and motif proxy features are stricter about returning `NaN`
  when no valid denominator exists
- older pitch and rhythm families still use explicit zero counts in some empty
  cases

That inconsistency is manageable, but anyone building downstream modeling or
comparative analytics should be aware of it.

## 17. Recommended Reading Order For Contributors

If you are onboarding into the project and want to understand the system
quickly, this is a useful reading order:

1. `representation/score_to_events.py`
2. `utils/music.py`
3. `features/base.py`
4. `features/registry.py`
5. the individual family extractors
6. `modeling/normalization.py`
7. `pipeline/extract.py`

That path follows the actual data flow and usually makes the design easier to
understand than reading the feature families in isolation.

