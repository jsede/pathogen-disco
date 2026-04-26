# Pathogen-Disco (v1.3)

A high-throughput, HPC-optimized bioinformatics pipeline for viromics, pathogen discovery, and complex metagenomic characterization from short-read sequencing data. 

Developed and maintained by the **Viromics @ WIMR** group (Westmead Institute for Medical Research). For more details on our research and other resources, visit [viromics.group](https://viromics.group).

## Overview
Pathogen-Disco is designed to efficiently process complex metagenomic samples (e.g., cloacal swabs, tissue samples, environmental samples) to identify and characterize viral and microbial pathogens. It handles everything from raw read QC to assembly, robust ORF-based annotation, and final taxonomic consensus, purposefully engineered for high-performance computing (HPC) clusters like NCI Gadi running PBS Pro.

### Key Features
* **HPC-Native Architecture:** Designed for PBS Pro with array/batch submission (`qsub`) and robust job scheduling.
* **Defensive Scripting:** Fully insulated against variable collisions, empty paths, and background system quirks.
* **ORF-Driven Annotation:** Utilizes Prodigal to extract Open Reading Frames (ORFs) from assemblies, ensuring long contigs (e.g., bulky bacterial or large viral fragments) are accurately annotated via rapid protein alignments.
* **"Winner-Takes-All" Taxonomy:** Employs a rigorous bitscore-first consensus algorithm to confidently assign taxonomy to assembled contigs based on their strongest, most conserved gene hits.
* **Unassembled Read Rescue:** Automatically catches sequences that fail assembly (like rare, low-titer viral fragments) and funnels them directly to Kraken2 for classification.

---

## Pipeline Workflow

The pipeline is split into six sequential modules:

### A. QC & Filtering (`pathogen-disco.v1.3-A-filter`)
* **Trimming & Deduplication:** Removes adapters, low-quality bases, and duplicates using `fastp`.
* **Complexity Filtering:** Filters out low-complexity sequences using `prinseq++` (DUST algorithm).
* **rRNA Depletion:** Rapidly sorts reads against SILVA/Rfam databases using `sortmerna`. rRNA reads are summarized using `kraken2` and `Krona`.
* **MetaCOXI Profiling:** Maps non-rRNA reads to the MetaCOXI database for baseline community profiling.
* **Host Removal:** Depletes host reads via `bowtie2` mapping against a target reference, followed by a secondary human read depletion using a `kraken2` HPRC database.

### B. Assembly & Quantification (`pathogen-disco.v1.3-B-assemble`)
* **De Novo Assembly:** Assembles clean, non-host reads using `megahit` (minimum contig length strictly set to 300bp to ensure robust downstream annotation).
* **ORF Extraction:** Predicts genes and translates proteins (`>= 50 AA`) using `prodigal` and `seqtk` for accurate protein-level searches.
* **Abundance Estimation:** Maps reads back to contigs using `kallisto` and `bowtie2` to determine TPM and coverage depth.
* **Unassembled Catch:** Gathers short fragments that failed to assemble and annotates them via `kraken2` (PlusPF database).

### C. Nucleotide Annotation (`pathogen-disco.v1.3-C-blast`)
* **AMR Gene Detection:** Screens non-host contigs for antimicrobial resistance genes using `amrfinder`.
* **Nucleotide Search:** Runs `blastn` against the NCBI `nt` database for standard sequence alignment.

### D. Protein Annotation (`pathogen-disco.v1.3-D-diamond`)
* **Rapid Translation Search:** Takes the `prodigal`-generated amino acid ORFs and runs `diamond blastp` against the clustered `nr` database. 

### E. Taxonomic Summarization (`pathogen-disco.v1.3-E-summary`)
* **Data Aggregation:** Calculates contig stats, merges Kallisto abundance (TPM), and remaps coverage.
* **Consensus Building:** Applies a highly-tuned "Winner-Takes-All" `awk` filter to select the absolute best taxonomic hit per contig (prioritizing highest bitscore, lowest e-value, then highest percent identity).
* **Lineage Formatting:** Pulls full taxonomic lineages using `taxonkit`.
* **Final Output:** Generates consolidated `.tsv` files and Top-10 summaries integrating abundance and taxonomy for effortless biological interpretation.

### F. Conserved Domain Search (`pathogen-disco.v1.3-F.orf-cdd`)
* **Functional Annotation:** Runs `rpsblast` against the NCBI Conserved Domain Database (CDD) to annotate specific protein domains within the predicted ORFs.

---

## Installation & Setup

### 1. Environment Management
The pipeline relies on `micromamba` (or `conda`) to handle all tool dependencies. 

Create the environment using the provided YAML file:
```bash
micromamba env create -f pathogen-disco.yml -p ${HOME}/environments/pathogen-disco
