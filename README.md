The SEQ-PC model is composed of four major systems: dataset builder, statistical normalization, the sequence modeling architecture (BiLSTM + Mixture-of-Experts), and the inference engine with stylistic shaping mechanisms.
1. High-Level Architecture

             ┌────────────────────┐
             │   Tuned UST Files  │
             └─────────┬──────────┘
                       │
                       ▼
             ┌────────────────────┐
             │   Dataset Builder  │
             │  (extract samples) │
             └─────────┬──────────┘
                       │ CSV + vocab
                       ▼
        ┌──────────────────────────────┐
        │  Statistical Normalization   │
        │ (compute target_stats.json)  │
        └───────────────┬───────────-──┘
                        │ normalized targets
                        ▼
         ┌─────────────────────────────┐
         │     SEQ-PC Model (BiLSTM    │
         │   + Mixture-of-Experts)     │
         └──────────────┬──────────────┘
                        │ predictions
                        ▼
        ┌──────────────────────────────┐
        │    Inference Engine          │
        │     (style shaping)          │
        └──────────────┬───────────-───┘
                       │ tuned pitch
                       ▼
              ┌──────────────────────┐
              │   Tuned Output UST   │
              └──────────────────────┘

2. Data Flow
 `  `  `
          UST File
             │
             ▼
          ┌──────────────────────────┐
          │ ust_parser               │
          │ - load notes             │
          │ - extract PBS/PBW/PBY    │
          │ - extract VBR            │
          │ - detect rests/phrases   │
          └───────────┬──────────────┘
                      │ samples
                      ▼
          ┌──────────────────────────┐
          │ dataset_builder.py       │
          │ - tempo, tempo_bucket    │
          │ - length_bucket          │
          │ - prev/next intervals    │
          │ - output → CSV row       │
          └───────────┬──────────────┘
                      │ CSV dataset
                      ▼
          info/dataset_notes.csv
 `  `  ` 

4. SEQ-PC Model Architecture
The model integrates a bidirectional LSTM encoder with a three-expert Mixture-of-Experts (MoE) output layer. The experts specialize in short, medium, and long note categories. A gating network assigns per-note expert weights.

 `  `  ` 
                 Sequence Input
     lyric ids ─────────────┐
 numeric features ──────────┘
                │
                ▼
        ┌───────────────────┐
        │  BiLSTM Encoder   │
        │  (2-layer, bi)    │
        └───────┬───────────┘
                │ 256-dim
                ▼
      ┌──────────────────────────────┐
      │        Gating Network        │
      │   produces weights w1,w2,w3  │
      └───────┬───────────┬─────────┘
              │           │
     ┌────────▼───┐   ┌───▼────────┐   ┌────────▼───┐
     │ Expert 1   │   │ Expert 2    │   │ Expert 3   │
     │ (short)    │   │ (medium)    │   │ (long)     │
     └─────┬──────┘   └────┬────────┘   └────┬──────┘
           │               │                 │
           └───────┬───────┴───────┬────────┘
                   ▼               ▼
             Weighted Sum of Experts
                        │
                        ▼
            Predicted Pitch Parameters
 `  `  ` 
5. Inference Pipeline
During inference,the model predicts pitch control parameters and then applies multiple stylistic shaping passes.
 `  `  ` 
UST Input
   │
   ▼
Feature Builder
   │ features
   ▼
SEQ-PC Model
   │ raw output
   ▼
Stylistic Shaping
   ├─ expressiveness scaling
   ├─ tuning amount blending
   ├─ phrase-start fry insertion
   ├─ long-note fixes
   ▼
UST Writer
   │ formatted PBS/PBW/PBY/VBR
   ▼
Final Tuned UST
 `  `  ` 
5. Post-Processing 
Post-processing stabilizes and stylizes the output so that the model's raw remains musically usable.
 `  `  ` 
Raw PBY Curve from Model
            │
            ▼
  ┌──────────────────────────────┐
  │ Expressiveness Scaling       │
  └───────────┬──────────────────┘
              │
              ▼
  ┌──────────────────────────────┐
  │ Curve Smoothing              │
  └───────────┬──────────────────┘
              │
              ▼
  ┌──────────────────────────────┐
  │ Long-Note Squish             │
  │ taper curve end section      │
  └───────────┬──────────────────┘
              │
              ▼
  ┌──────────────────────────────┐
  │ Phrase-Start Fry Override    │
  │ insert fry → zero dip        │
  └───────────┬──────────────────┘
              │ final curves
              ▼
    Output PBY for UST
 `  `  ` 
