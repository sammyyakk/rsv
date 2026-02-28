# Product / Project Requirements Document (PRD)

## Project Title
AI-Assisted Drug Discovery for Respiratory Syncytial Virus (RSV) via Molecular Docking Simulations

---

## 1. Project Overview

This project applies AI-assisted virtual screening and open-source molecular docking tools to identify candidate drugs—both existing (repurposed) and newly proposed—that can bind effectively to key RSV protein targets. By leveraging AutoDock Vina (and compatible FOSS platforms), the project aims to accelerate the early-stage drug discovery pipeline for RSV without requiring wet-lab resources in the initial phase.

---

## 2. Objectives

| # | Objective |
|---|-----------|
| 1 | Identify and prepare 3-D structures of validated RSV protein targets (from PDB). |
| 2 | Prepare ligand structures for all candidate drugs (standard and test). |
| 3 | Perform automated molecular docking simulations for each drug–target pair. |
| 4 | Rank candidates by binding affinity and visualise binding poses. |
| 5 | Generate a written research report suitable for academic publication. |

---

## 3. Scope

### 3.1 In-Scope
- Virtual screening of the drugs listed in `RSV Drugs.docx` (see §6).
- Protein targets drawn from publicly available PDB entries referenced in that document.
- Automated docking pipeline using AutoDock Vina / Smina / DiffDock (all free & open-source).
- AI-assisted analysis of docking scores and binding-pocket residues.
- Automated report generation (PDF/Markdown) summarising findings.

### 3.2 Out-of-Scope
- Wet-lab experimental validation.
- Molecular dynamics (MD) simulations (may be added in a future phase).
- Novel drug design / de-novo generation (may be added in a future phase).
- Clinical or regulatory submissions.

---

## 4. Stakeholders

| Role | Responsibility |
|------|---------------|
| Principal Researcher | Scientific direction, paper authorship |
| AI Coding Agent | Pipeline implementation, automation |
| Computational Chemistry Reviewer | Validate docking parameters and results |

---

## 5. Success Criteria

1. All 13 drug–target pairs complete docking without errors.
2. Binding-affinity scores (kcal/mol) are produced and ranked for every pair.
3. At least one repurposed test drug shows a predicted binding affinity comparable to or better than a standard RSV drug (Ribavirin / Nirsevimab / Palivizumab).
4. A structured research report is automatically generated.
5. All code, inputs, and outputs are reproducible from a single command.

---

## 6. Drug & Target Inventory

The following drugs and PDB IDs are sourced from `RSV Drugs.docx`:

| Drug | PDB ID | Category |
|------|--------|----------|
| Ribavirin | 4PB1 | Standard Drug |
| Nirsevimab | 5UDC | Standard Drug |
| Palivizumab | 5J3D | Standard Drug |
| Nitazoxanide | 3V36 | Test Drug |
| Azithromycin | 4V7Y | Test Drug |
| Hydroxychloroquine | 8S5Z | Test Drug |
| Atorvastatin | 1HWK | Test Drug |
| Resveratrol | 4Q93 | Test Drug |
| Simvastatin | 1HW9 | Test Drug |
| Celecoxib | 3LN1 | Test Drug |
| Bortezomib | 5LF3 | Test Drug |
| Minocycline | 9FE2 | Test Drug |
| Favipiravir | 7AAP | Test Drug |

---

## 7. Functional Requirements

| ID | Requirement |
|----|-------------|
| FR-01 | The system SHALL download PDB structures automatically using RCSB PDB REST API. |
| FR-02 | The system SHALL fetch 2-D ligand structures from PubChem and convert them to 3-D using RDKit or Open Babel. |
| FR-03 | The system SHALL prepare receptor PDB files (remove water, add polar H, assign Gasteiger charges) using AutoDockTools / ADFR Suite / Meeko. |
| FR-04 | The system SHALL define docking search boxes automatically around the native co-crystal ligand binding pocket. |
| FR-05 | The system SHALL run AutoDock Vina (or Smina / DiffDock) for each drug–receptor pair. |
| FR-06 | The system SHALL extract the best-pose binding affinity (kcal/mol) and RMSD values. |
| FR-07 | The system SHALL produce a ranked results table in CSV and Markdown. |
| FR-08 | The system SHALL generate 3-D binding-pose visualisations (PyMOL / py3Dmol). |
| FR-09 | The system SHALL compile a research-ready PDF report using a LaTeX or Markdown-to-PDF template. |
| FR-10 | All pipeline steps SHALL be logged and reproducible via a single entry-point script (`run_pipeline.py`). |

---

## 8. Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NFR-01 | All tools used must be free and open-source. |
| NFR-02 | The pipeline must run on a standard Linux/macOS workstation (≥8 GB RAM, GPU optional). |
| NFR-03 | End-to-end runtime for all 13 compounds must be ≤4 hours on CPU-only hardware. |
| NFR-04 | Code must be written in Python 3.10+ and follow PEP 8. |
| NFR-05 | Results must be version-controlled in this repository. |
| NFR-06 | Intermediate files (PDB, PDBQT, log) must be organised in a well-defined directory structure. |

---

## 9. Constraints & Assumptions

- Internet access is required to download PDB structures and ligand SMILES strings.
- The PDB IDs in `RSV Drugs.docx` point to structures containing the target protein; preprocessing will extract the relevant chain and remove co-crystal ligands before re-docking.
- AutoDock Vina 1.2+ is assumed to be installable via conda or apt.

---

## 10. Glossary

| Term | Definition |
|------|-----------|
| PDB | Protein Data Bank – repository of 3-D macromolecular structures. |
| PDBQT | PDB format with partial charges and atom types used by AutoDock. |
| Binding Affinity | Free energy of binding (kcal/mol); more negative = stronger predicted binding. |
| Virtual Screening | Computational method to evaluate large sets of ligands against a target. |
| Docking | Computational prediction of the preferred orientation of a ligand in a receptor binding site. |
| RSV | Respiratory Syncytial Virus – a common respiratory pathogen. |
| FOSS | Free and Open-Source Software. |
