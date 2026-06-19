# RTL Agent Pipeline v3

**Text / PDF / DOCX / Audio / Video → Spec IR → Two-Oracle Agent → RTL**

End-to-end automated RTL generation from multiple input sources via a Two-Oracle agent with spec-derived conformance checking and action-constrained repair. Supports **multi-source input**: combine text, PDF, DOCX, audio, and video in a single run.


## Quick Start

```bash
pip install -r requirements.txt
export GOOGLE_API_KEY=your_key

# Interactive (multi-source: text, PDF, DOCX, audio, video)
python run_local.py

# One-shot from CLI (single source)
python cli.py --text "8-bit multiplier"
python cli.py --pdf spec.pdf --prompt "Focus on Figure 2"
python cli.py --docx requirements.docx
python cli.py --audio walkthrough.mp3
python cli.py --video architecture_walkthrough.mp4

# Multi-source (combine any mix, flags are repeatable)
python cli.py --text "priority: minimize area" --pdf datasheet.pdf --audio narration.mp3 --video design_review.mp4

# Benchmark
python cli.py --benchmark
```

---

## Supported Input Types

| Type | Flag / Option | Notes |
|------|--------------|-------|
| Text | `--text "…"` | Freeform hardware description |
| PDF | `--pdf path.pdf` | Research paper, datasheet (text + images) |
| DOCX | `--docx path.docx` | Word document (text + tables) |
| Audio | `--audio path.mp3` | Spoken requirement / design walkthrough |
| Video | `--video path.mp4` | Walkthrough/demo/lecture video (transcript + requirement extraction) |

- All flags are **repeatable** (e.g. `--pdf p1.pdf --pdf p2.pdf`).
- When multiple sources are provided they are merged into a single evidence bundle before canonicalization.
- Video and audio are first converted to extracted textual evidence, then canonicalized.
- In interactive mode (`run_local.py`) you are prompted after each source to add more.

---

## Embeddings (Optional Retrieval Layer)

`tools/embeddings.py` exposes a lightweight Gemini-embedding–based ranking layer:

```python
from tools.embeddings import select_relevant_evidence, rank_chunks_by_query

# Filter a large evidence bundle to the most relevant chunks for a query
filtered = select_relevant_evidence(evidence_bundle, user_prompt="Focus on the FSM", client=genai)

# Rank arbitrary text chunks by relevance to a query
top_chunks = rank_chunks_by_query(chunks, query="clock domain crossing", client=genai, top_k=8)
```

This layer is **not** on the critical path — if unused, the full evidence text is always passed to the canonicalizer unchanged.

---

## Pipeline Architecture

```
User Input(s)
  │  text / PDF / DOCX / audio / video
  ▼
InputCollector  (input_layer.py)
  │  extract_from_* + collect_and_extract
  ▼
Evidence Bundle
  │  combined_text + images + source provenance
  │  [optional: EmbeddingsRanker trims to top-K chunks]
  ▼
Canonicalizer  (spec/canonicalizer.py)
  │  → Spec IR (JSON)
  ▼
Two-Oracle Pipeline  (pipeline.py)
  │  Writer → Verilator → Icarus (spec_tb / llm_tb)
  │  Controller → Reviewer (FIX_PARSE|FIX_WIDTH|FIX_FUNCTION…)
  ▼
Post-Pass Backend (optional)
  │  Yosys synthesis + ABC  ·  OpenSTA  ·  SymbiYosys formal
  ▼
Outputs: final_dut.sv · final_tb.sv · circuit.svg · JSONL log
```

---

## Project Structure

```
agent/
├── config.py              # API key, model names, paths
├── cli.py                 # CLI entry (text, PDF, DOCX, audio, video, benchmark)
├── run_local.py           # Interactive multi-source local runner
├── pipeline.py            # Main Two-Oracle loop
├── input_layer.py         # All input extraction + evidence aggregation
├── controller.py          # Failure classification → Diagnosis
├── utils.py               # Gemini API wrapper with retry/backoff
├── spec/
│   ├── schema.py          # Spec IR schema + validation
│   ├── canonicalizer.py   # LLM → Spec IR (text / PDF / mixed)
│   └── test_generator.py  # Spec IR → deterministic testbench
├── agents/
│   ├── writer.py          # RTL + TB generation
│   └── reviewer.py        # Targeted action-constrained repair
├── tools/
│   ├── verilator.py       # Lint (Verilator)
│   ├── simulator.py       # Icarus Verilog compile + sim
│   ├── synthesis.py       # Yosys synthesis, stat, mapping
│   ├── visualizer.py      # Circuit SVG (Yosys show)
│   ├── metrics.py         # Parse Yosys stat output
│   ├── formal.py          # SymbiYosys (optional)
│   ├── sta.py             # OpenSTA (optional)
│   └── embeddings.py      # Gemini embeddings – chunk ranking (optional)
├── eval/
│   ├── benchmark.py       # Benchmark runner
│   └── paper_specs/       # Sample Spec IR JSON files
└── rtl_agent_pipeline.ipynb   # Google Colab notebook
```

---

## Outputs

| Output | Location | Description |
|--------|----------|-------------|
| `final_dut.sv` | work_dir | Generated RTL |
| `final_tb.sv` | work_dir | Final testbench |
| `final_gate.v` | work_dir | Gate-level netlist (if synthesis ran) |
| `circuit.svg` | work_dir | Circuit schematic diagram |
| `iterations.jsonl` | work_dir | Per-iteration log with oracle confidence |
| `benchmark_report.json` | eval/results | Benchmark summary |

---

## CLI Reference

```bash
python cli.py                                   # Interactive mode
python cli.py --text "2-input AND gate"
python cli.py --pdf spec.pdf --prompt "Implement Table 1"
python cli.py --docx requirements.docx
python cli.py --audio narration.mp3 --prompt "Focus on counter design"
python cli.py --video walkthrough.mp4 --prompt "Focus on FSM segment"
python cli.py --pdf a.pdf --pdf b.pdf --text "add timing constraint"
python cli.py --benchmark --specs-dir eval/paper_specs
python cli.py --formal                          # Enable SymbiYosys
python cli.py --verbose                         # Print LLM I/O
```

---

## Google Colab

1. Open `rtl_agent_pipeline.ipynb` in Google Colab
2. Run **Cell 1**: Mount Drive + install `iverilog`, `verilator`, `yosys`, `graphviz`, `python-docx`
3. Run **Cell 2**: Add `GEMINI_API_KEY` in Colab Secrets
4. Run **Cell 3**: Load pipeline modules
5. Run **Cell 4**: Add sources (text / PDF / DOCX / audio / video), then run the agent
6. Run **Cell 5**: View results (SVG, RTL, testbench)

---

## Tools (open-source, free)

| Tool | Role |
|------|------|
| **Verilator** | Syntax / semantic lint |
| **Icarus Verilog** | Simulation (spec TB + LLM TB) |
| **Yosys** | Synthesis, area report, gate-level netlist |
| **Graphviz** | Circuit SVG via `yosys show` |
| **SymbiYosys** | Formal verification / BMC (optional, `--formal`) |
| **OpenSTA** | Static timing analysis: WNS/TNS (optional) |
| **PyMuPDF** | PDF text + image extraction |
| **Pillow** | PIL image processing for multimodal input |
| **python-docx** | DOCX text + table extraction |

---

## API

Uses **Gemini 2.5 Flash-Lite** (generation) and **gemini-embedding-001** (embeddings, optional).  
Set `GOOGLE_API_KEY` or `GEMINI_API_KEY`.

Production toggles:
- `RTL_USE_EMBEDDINGS=1` enables retrieval-ranking before canonicalization (default: enabled).
- `RTL_EMBED_TOP_K=8` controls how many ranked chunks are kept.

---

## Benchmark Results

Evaluated on our internal RTL generation benchmark suite (`python cli.py --benchmark`). Comparisons against published baselines on VerilogEval v2 (syntax-correct + functional pass@1).

| System | pass@1 (functional) | Self-repair | Multi-modal input |
|--------|--------------------:|:-----------:|:-----------------:|
| GPT-4 (VerilogEval v2 baseline) | ~53 % | ✗ | ✗ |
| MAGE | ~61 % | ✓ | ✗ |
| AIvril2 | ~67 % | ✓ | ✗ |
| **RTL Agent v3 (ours)** | **TODO** | ✓ | ✓ |

> **Note:** Full benchmark numbers are being finalized for the GLSVLSI 2026 camera-ready submission. Check back for updated figures. Baseline numbers sourced from published papers; see [VerilogEval v2](https://arxiv.org/abs/2406.xxxxx), [MAGE](https://arxiv.org/abs/2405.xxxxx), [AIvril2](https://arxiv.org/abs/2406.xxxxx).

Key differentiators vs. baselines:
- **Multi-source input** — text, PDF, DOCX, audio, video fused before spec extraction (no prior work combines all five)
- **Spec IR conformance oracle** — deterministic spec-derived testbench catches functional regressions baselines miss
- **Action-constrained repair** — controller classifies failure type (parse / width / functional) and issues targeted fix actions rather than blind regeneration
