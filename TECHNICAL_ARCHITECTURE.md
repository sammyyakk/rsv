# Technical Architecture Specification

## Project: AI-Assisted RSV Drug Discovery via Molecular Docking

---

## 1. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        run_pipeline.py                              │
│                     (single entry-point)                            │
└────────────┬───────────────────────────────────────────────────────┘
             │
    ┌────────▼────────┐    ┌──────────────────┐    ┌───────────────┐
    │  1. Data        │    │  2. Ligand        │    │  3. Receptor  │
    │  Acquisition    │───▶│  Preparation      │───▶│  Preparation  │
    │  (PDB + SMILES) │    │  (3-D + charges)  │    │  (clean PDB)  │
    └─────────────────┘    └──────────────────┘    └──────┬────────┘
                                                          │
    ┌─────────────────┐    ┌──────────────────┐    ┌──────▼────────┐
    │  6. Report      │    │  5. Analysis &   │    │  4. Docking   │
    │  Generation     │◀───│  Visualisation   │◀───│  (Vina/Smina) │
    │  (PDF/Markdown) │    │  (scores + poses)│    │               │
    └─────────────────┘    └──────────────────┘    └───────────────┘
```

---

## 2. Technology Stack

| Layer | Tool / Library | Version | Purpose |
|-------|---------------|---------|---------|
| Language | Python | 3.10+ | Pipeline orchestration |
| Cheminformatics | RDKit | ≥2023.03 | SMILES → 3-D SDF, conformer generation |
| Format conversion | Open Babel (obabel) | ≥3.1 | SDF ↔ PDBQT conversion |
| Receptor prep | Meeko (ADFRsuite) | ≥0.5 | PDB → PDBQT, polar-H, Gasteiger charges |
| Docking engine | AutoDock Vina | 1.2+ | Molecular docking simulations |
| Alternative engine | Smina / DiffDock | latest | GPU-accelerated or deep-learning docking |
| Structure download | Biopython / RCSB API | ≥1.81 | PDB file retrieval |
| Ligand SMILES | PubChemPy | ≥1.0.4 | PubChem REST API wrapper |
| Data handling | Pandas | ≥2.0 | Results tables, CSV output |
| Visualisation | py3Dmol / PyMOL | latest | 3-D binding pose images |
| Reporting | Jinja2 + WeasyPrint | latest | HTML → PDF report |
| Dependency mgmt | Conda + pip | latest | Environment reproducibility |

---

## 3. Directory Structure

```
rsv/
├── PRD.md                          # Product Requirements Document
├── TECHNICAL_ARCHITECTURE.md       # This document
├── EXECUTION_PLAN.md               # Agent task plan
├── RSV Drugs.docx                  # Original drug/PDB reference
├── run_pipeline.py                 # Single entry-point script
├── config/
│   └── drugs.yaml                  # Drug list + PDB IDs (machine-readable)
├── src/
│   ├── data_acquisition.py         # Download PDB structures + SMILES
│   ├── ligand_prep.py              # SMILES → 3-D → PDBQT
│   ├── receptor_prep.py            # Clean PDB → PDBQT
│   ├── docking.py                  # AutoDock Vina wrapper
│   ├── analysis.py                 # Parse results, rank, visualise
│   └── report.py                   # Generate PDF/Markdown report
├── data/
│   ├── raw/                        # Downloaded PDB files (.pdb)
│   ├── receptors/                  # Prepared receptor files (.pdbqt)
│   ├── ligands/                    # Prepared ligand files (.pdbqt)
│   └── docking/                    # Vina output files (.pdbqt, .log)
├── results/
│   ├── scores.csv                  # Consolidated binding affinity table
│   ├── ranked_results.md           # Human-readable ranked table
│   ├── poses/                      # Best-pose images (PNG)
│   └── report.pdf                  # Final research report
├── templates/
│   └── report_template.html        # Jinja2 HTML template for report
├── environment.yml                 # Conda environment specification
└── requirements.txt                # pip requirements (non-conda deps)
```

---

## 4. Data Flow

### 4.1 Drug Configuration (`config/drugs.yaml`)

```yaml
drugs:
  - name: Ribavirin
    pdb_id: 4PB1
    category: standard
  - name: Nirsevimab
    pdb_id: 5UDC
    category: standard
  # ... (all 13 entries)
```

### 4.2 Module Interfaces

#### `data_acquisition.py`
```
Input:  drugs.yaml
Output: data/raw/<PDB_ID>.pdb
        data/ligands/<drug_name>.smi  (SMILES string from PubChem)
```

#### `ligand_prep.py`
```
Input:  data/ligands/<drug_name>.smi
Steps:  1. Parse SMILES with RDKit
        2. Generate 3-D conformer (ETKDG)
        3. Minimise geometry (MMFF94)
        4. Convert to PDBQT via Meeko (polar-H, Gasteiger charges)
Output: data/ligands/<drug_name>.pdbqt
```

#### `receptor_prep.py`
```
Input:  data/raw/<PDB_ID>.pdb
Steps:  1. Extract target chain (Biopython PDBParser)
        2. Remove HETATM (water, co-crystal ligands)
        3. Add polar hydrogens (reduce / pdbfixer)
        4. Assign Gasteiger charges & convert to PDBQT (Meeko)
        5. Auto-detect binding box from original HETATM coordinates
Output: data/receptors/<PDB_ID>_receptor.pdbqt
        data/receptors/<PDB_ID>_box.json   # centre_x/y/z, size_x/y/z
```

#### `docking.py`
```
Input:  receptor PDBQT, ligand PDBQT, box JSON
Steps:  1. Build AutoDock Vina command
        2. Run subprocess (exhaustiveness=16, num_modes=9)
        3. Capture stdout / log
Output: data/docking/<PDB_ID>_<drug>_out.pdbqt
        data/docking/<PDB_ID>_<drug>.log
```

#### `analysis.py`
```
Input:  all docking logs
Steps:  1. Parse best binding affinity (mode 1) from each log
        2. Build Pandas DataFrame
        3. Rank by affinity (ascending kcal/mol)
        4. Generate 3-D pose images via py3Dmol (exported PNG)
Output: results/scores.csv
        results/ranked_results.md
        results/poses/<pair>.png
```

#### `report.py`
```
Input:  results/scores.csv, results/poses/, templates/report_template.html
Steps:  1. Render Jinja2 template with data
        2. Convert HTML → PDF via WeasyPrint
Output: results/report.pdf
```

---

## 5. Docking Parameters

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Exhaustiveness | 16 | Balance of speed vs. thoroughness on CPU |
| Number of modes | 9 | Standard; captures top binding poses |
| Energy range | 3 kcal/mol | Keeps near-optimal poses |
| Search box padding | 10 Å on each side around pocket centre | Covers full binding site |
| Scoring function | AutoDock Vina (Vina score) | Default; well-validated |

---

## 6. AI Integration Points

| Step | AI Role |
|------|---------|
| Binding-pocket detection | If no co-crystal ligand: use `fpocket` or a ML-based pocket predictor (P2Rank) |
| Ligand SMILES retrieval | PubChem API (deterministic); LLM fallback for names not found |
| Results interpretation | GPT-4 / local LLM generates natural-language summary of binding interactions |
| Report writing | LLM drafts Introduction, Methods, Results, Discussion sections seeded with docking data |

---

## 7. Error Handling & Logging

- Every module writes to a rotating log file (`pipeline.log`) via Python `logging`.
- Failed docking jobs are retried once; persistent failures are flagged in the results table with `N/A`.
- All subprocess calls capture `stderr`; non-zero exit codes raise `DockingError`.

---

## 8. Environment Setup

### Conda (recommended)

```bash
conda env create -f environment.yml
conda activate rsv-docking
```

### Key conda packages
```yaml
channels:
  - conda-forge
  - bioconda
dependencies:
  - python=3.10
  - rdkit
  - openbabel
  - vina          # AutoDock Vina 1.2
  - biopython
  - pandas
  - py3dmol
  - weasyprint
  - jinja2
  - pip:
    - meeko
    - pubchempy
    - pdbfixer
```

---

## 9. Reproducibility

- `environment.yml` pins all package versions.
- Random seeds are fixed (`random.seed(42)`, `numpy.random.seed(42)`).
- All downloaded PDB files are cached; re-runs skip downloads unless `--refresh` flag is passed.
- Results are committed to the repository in `results/`.

---

## 10. Security & Licensing

| Concern | Mitigation |
|---------|-----------|
| All tools FOSS | AutoDock Vina (Apache 2.0), RDKit (BSD), Open Babel (GPL-2), Biopython (BSD) |
| No proprietary data | All structures from public PDB; ligand SMILES from PubChem |
| No credentials stored | API calls are unauthenticated (RCSB & PubChem are public) |
