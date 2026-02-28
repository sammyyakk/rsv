# Execution / Task Plan — Agent Instructions

## Project: AI-Assisted RSV Drug Discovery via Molecular Docking

> **For AI Agents:** Follow each phase sequentially. Complete all checklist items before moving to the next phase. Log progress to `pipeline.log`. On any unrecoverable error, halt and report the last successful step.

---

## Phase 0 — Environment Bootstrap

**Goal:** Ensure all dependencies are installed and the workspace is ready.

### Tasks

- [ ] **T-0.1** Clone/verify the repository at the working directory root.
- [ ] **T-0.2** Create and activate the conda environment from `environment.yml`:
  ```bash
  conda env create -f environment.yml
  conda activate rsv-docking
  ```
- [ ] **T-0.3** Verify tool availability:
  ```bash
  python -c "import rdkit, meeko, pubchempy, biopython; print('OK')"
  vina --version
  obabel --version
  ```
- [ ] **T-0.4** Create the output directory tree:
  ```
  data/raw/  data/receptors/  data/ligands/  data/docking/
  results/scores/  results/poses/
  ```
- [ ] **T-0.5** Write `config/drugs.yaml` (machine-readable version of `RSV Drugs.docx` table):

  ```yaml
  drugs:
    - name: Ribavirin
      pdb_id: 4PB1
      category: standard
    - name: Nirsevimab
      pdb_id: 5UDC
      category: standard
    - name: Palivizumab
      pdb_id: 5J3D
      category: standard
    - name: Nitazoxanide
      pdb_id: 3V36
      category: test
    - name: Azithromycin
      pdb_id: 4V7Y
      category: test
    - name: Hydroxychloroquine
      pdb_id: 8S5Z
      category: test
    - name: Atorvastatin
      pdb_id: 1HWK
      category: test
    - name: Resveratrol
      pdb_id: 4Q93
      category: test
    - name: Simvastatin
      pdb_id: 1HW9
      category: test
    - name: Celecoxib
      pdb_id: 3LN1
      category: test
    - name: Bortezomib
      pdb_id: 5LF3
      category: test
    - name: Minocycline
      pdb_id: 9FE2
      category: test
    - name: Favipiravir
      pdb_id: 7AAP
      category: test
  ```

**Exit criteria:** All tools respond without error; directory tree created; `config/drugs.yaml` written.

---

## Phase 1 — Data Acquisition

**Goal:** Download all PDB structures and retrieve SMILES strings for all ligands.

**Module:** `src/data_acquisition.py`

### Tasks

- [ ] **T-1.1** For **each entry** in `config/drugs.yaml`, download the PDB file from RCSB:
  ```
  https://files.rcsb.org/download/<PDB_ID>.pdb
  ```
  Save to `data/raw/<PDB_ID>.pdb`. Skip if file already exists.

- [ ] **T-1.2** For **each drug name**, fetch the canonical SMILES from PubChem:
  ```python
  import pubchempy as pcp
  compounds = pcp.get_compounds(drug_name, 'name')
  smiles = compounds[0].isomeric_smiles
  ```
  Save to `data/ligands/<drug_name>.smi`. Log a warning if not found.

- [ ] **T-1.3** Validate that all 13 PDB files were downloaded successfully.
- [ ] **T-1.4** Validate that all 13 SMILES strings were retrieved.

**Exit criteria:** `data/raw/` contains 13 `.pdb` files; `data/ligands/` contains 13 `.smi` files.

---

## Phase 2 — Ligand Preparation

**Goal:** Convert each SMILES string into a 3-D PDBQT file suitable for AutoDock Vina.

**Module:** `src/ligand_prep.py`

### Tasks

For **each drug**, perform the following steps:

- [ ] **T-2.1** Parse SMILES with RDKit:
  ```python
  from rdkit import Chem
  from rdkit.Chem import AllChem
  mol = Chem.MolFromSmiles(smiles)
  mol = Chem.AddHs(mol)
  ```

- [ ] **T-2.2** Generate a 3-D conformer using ETKDG:
  ```python
  AllChem.EmbedMolecule(mol, AllChem.ETKDGv3())
  ```

- [ ] **T-2.3** Minimise geometry with MMFF94:
  ```python
  AllChem.MMFFOptimizeMolecule(mol)
  ```

- [ ] **T-2.4** Write to SDF: `data/ligands/<drug_name>.sdf`

- [ ] **T-2.5** Convert SDF → PDBQT using Meeko:
  ```bash
  mk_prepare_ligand.py -i data/ligands/<drug_name>.sdf \
                       -o data/ligands/<drug_name>.pdbqt
  ```

- [ ] **T-2.6** Validate PDBQT file is non-empty and contains `ATOM` records.

**Exit criteria:** `data/ligands/` contains 13 `.pdbqt` files.

---

## Phase 3 — Receptor Preparation

**Goal:** Prepare each PDB receptor file for docking (clean → add H → PDBQT) and detect the binding box.

**Module:** `src/receptor_prep.py`

### Tasks

For **each PDB entry**, perform the following steps:

- [ ] **T-3.1** Parse the PDB file with Biopython `PDBParser`.

- [ ] **T-3.2** Identify the largest protein chain (ATOM records only).

- [ ] **T-3.3** Record the centroid of all HETATM (co-crystal ligand / small molecule) coordinates for binding-box definition. Store in `data/receptors/<PDB_ID>_box.json`:
  ```json
  {
    "center_x": 0.0, "center_y": 0.0, "center_z": 0.0,
    "size_x": 20.0,  "size_y": 20.0,  "size_z": 20.0
  }
  ```
  If no HETATM is present, use `fpocket` or P2Rank to predict the pocket.

- [ ] **T-3.4** Remove water molecules (HOH residues) and all HETATM records.

- [ ] **T-3.5** Add polar hydrogens using `pdbfixer` or `reduce`:
  ```python
  from pdbfixer import PDBFixer
  fixer = PDBFixer(filename=pdb_path)
  fixer.addMissingHydrogens(7.4)
  ```

- [ ] **T-3.6** Convert cleaned PDB → PDBQT using Meeko:
  ```bash
  mk_prepare_receptor.py -i data/receptors/<PDB_ID>_clean.pdb \
                         -o data/receptors/<PDB_ID>_receptor.pdbqt
  ```

- [ ] **T-3.7** Validate PDBQT file is non-empty.

**Exit criteria:** `data/receptors/` contains 13 `_receptor.pdbqt` files and 13 `_box.json` files.

---

## Phase 4 — Molecular Docking

**Goal:** Run AutoDock Vina for every drug–receptor pair and capture docking scores.

**Module:** `src/docking.py`

### Tasks

- [ ] **T-4.1** For **each (drug, PDB_ID) pair** (13 pairs), build and execute the Vina command:
  ```bash
  vina \
    --receptor data/receptors/<PDB_ID>_receptor.pdbqt \
    --ligand   data/ligands/<drug_name>.pdbqt \
    --center_x <cx> --center_y <cy> --center_z <cz> \
    --size_x   <sx> --size_y   <sy> --size_z   <sz> \
    --exhaustiveness 16 \
    --num_modes 9 \
    --energy_range 3 \
    --out  data/docking/<PDB_ID>_<drug_name>_out.pdbqt \
    --log  data/docking/<PDB_ID>_<drug_name>.log
  ```

- [ ] **T-4.2** Capture and persist stdout/stderr to the log file.
- [ ] **T-4.3** On non-zero exit code, retry once. On second failure, record `N/A` for this pair in the results table and continue.
- [ ] **T-4.4** Confirm all 13 log files exist after this phase.

**Exit criteria:** `data/docking/` contains log files for all 13 pairs (or `N/A` entries noted).

---

## Phase 5 — Analysis & Scoring

**Goal:** Parse docking results, rank candidates, and generate visualisations.

**Module:** `src/analysis.py`

### Tasks

- [ ] **T-5.1** For **each log file**, extract the best (mode 1) binding affinity:
  ```
  Pattern in log: "   1      -X.XX      ..."
  ```
  Record: `drug_name`, `pdb_id`, `category`, `affinity_kcal_mol`, `rmsd_lb`, `rmsd_ub`.

- [ ] **T-5.2** Assemble all records into a Pandas DataFrame.

- [ ] **T-5.3** Sort ascending by `affinity_kcal_mol` (most negative = best binder).

- [ ] **T-5.4** Save to `results/scores.csv`.

- [ ] **T-5.5** Write `results/ranked_results.md` with a formatted Markdown table:
  ```
  | Rank | Drug | PDB ID | Category | Affinity (kcal/mol) |
  ```

- [ ] **T-5.6** For the **top 5** ranked drugs, generate a 3-D binding-pose image using py3Dmol and save to `results/poses/<drug_name>.png`.

- [ ] **T-5.7** Compute and log the mean affinity of standard drugs vs. test drugs for comparison.

**Exit criteria:** `results/scores.csv` exists; `results/ranked_results.md` exists; top-5 PNG images exist.

---

## Phase 6 — Report Generation

**Goal:** Produce a research-ready PDF report.

**Module:** `src/report.py`

### Tasks

- [ ] **T-6.1** Load `results/scores.csv` and `results/ranked_results.md`.

- [ ] **T-6.2** Render `templates/report_template.html` via Jinja2, injecting:
  - Project title, date, author placeholder
  - Methods section (docking parameters from config)
  - Results table (from scores.csv)
  - Top-5 binding-pose images
  - Discussion: highlight any test drug that outperforms standard drugs

- [ ] **T-6.3** Convert rendered HTML → PDF with WeasyPrint:
  ```python
  from weasyprint import HTML
  HTML(string=html_content).write_pdf('results/report.pdf')
  ```

- [ ] **T-6.4** Validate `results/report.pdf` is non-empty (> 1 KB).

- [ ] **T-6.5** (Optional) Also save Markdown version as `results/report.md` for version control.

**Exit criteria:** `results/report.pdf` generated and validated.

---

## Phase 7 — Final Validation & Commit

**Goal:** Ensure full reproducibility and commit results.

### Tasks

- [ ] **T-7.1** Run the full pipeline from scratch in a clean directory to confirm end-to-end reproducibility:
  ```bash
  python run_pipeline.py --refresh
  ```

- [ ] **T-7.2** Compare `results/scores.csv` checksums between two runs (must be identical for same inputs).

- [ ] **T-7.3** Confirm that `pipeline.log` records timestamps for every phase and every compound.

- [ ] **T-7.4** Commit `results/scores.csv`, `results/ranked_results.md`, and `results/report.pdf` to the repository.

- [ ] **T-7.5** Write a brief `RESULTS_SUMMARY.md`:
  - Best-binding test drug vs. best standard drug
  - Key binding residues if identified
  - Next recommended steps (MD simulation, synthesis priority)

**Exit criteria:** All results committed; `RESULTS_SUMMARY.md` written.

---

## Error Recovery Reference

| Error | Recovery Action |
|-------|----------------|
| PDB download fails | Retry with 5 s back-off ×3; then skip and log |
| PubChem SMILES not found | Try synonym lookup; if still missing, use a known SMILES from ChEMBL |
| Conformer generation fails | Try `AllChem.EmbedMolecule` with `randomSeed=42`; if still failing, use `ETKDGv2` |
| Meeko PDBQT conversion fails | Fall back to `obabel -ipdb ... -opdbqt ...` |
| Vina returns no valid poses | Enlarge search box by 5 Å; retry |
| report.pdf empty | Check WeasyPrint HTML; fallback to pandoc Markdown→PDF |

---

## Deliverables Checklist

| Deliverable | Location | Status |
|-------------|----------|--------|
| Drug config (YAML) | `config/drugs.yaml` | - [ ] |
| Raw PDB files | `data/raw/*.pdb` | - [ ] |
| Ligand PDBQT files | `data/ligands/*.pdbqt` | - [ ] |
| Receptor PDBQT files | `data/receptors/*.pdbqt` | - [ ] |
| Docking output files | `data/docking/*.pdbqt` | - [ ] |
| Docking log files | `data/docking/*.log` | - [ ] |
| Scores CSV | `results/scores.csv` | - [ ] |
| Ranked results | `results/ranked_results.md` | - [ ] |
| Pose images (top 5) | `results/poses/*.png` | - [ ] |
| Research report PDF | `results/report.pdf` | - [ ] |
| Results summary | `RESULTS_SUMMARY.md` | - [ ] |
