# Spectral-library-based bottom-up proteomics with DIA-NN

A hands-on tutorial for spectral-library-based bottom-up proteomics using DIA-NN, built around the tear-fluid proteomics study:

**Ahmed, S., Altman, J., Jones, G., Mayernik, D., Williams, E., Mahmoud, A., Lee, T.J., Zhi, W., Sharma, S. and Sharma, A. (2025). _Tear fluid proteomics: a comparative study of DIA and DDA mass spectrometry_. Journal of Mass Spectrometry and Advances in the Clinical Lab.**

This tutorial works with the public PRIDE datasets linked to the publication:

- **PXD062423** — DIA raw files, DIA-NN reports, pooled DIA replicates, and DIA dilution-series files.
- **PXD062422** — DDA raw files, pooled DDA replicates, and DDA dilution-series files.

The goal is to reproduce the logic of the publication by comparing **DIA** and **DDA** input files from the same tear-fluid proteomics experiment. The DIA files are processed with DIA-NN using predicted spectral libraries generated from FASTA databases. The DDA files can be processed separately, for example with FragPipe or Proteome Discoverer, and then compared with the DIA-NN output at the protein and peptide level.

The workflow goes from raw file download through FASTA preparation, Docker setup, `.raw` to `.mzML` conversion, DIA-NN spectral-library generation, DIA-NN analysis, and downstream R-based comparison of DIA and DDA results.

---

## What's covered

- Downloading DIA raw files and DIA-NN report tables from PRIDE project **PXD062423**
- Downloading the DDA companion raw files from PRIDE project **PXD062422**
- Separating pooled replicate files from dilution-series files
- Downloading reviewed UniProt Swiss-Prot FASTA files for human and *E. coli*
- Building a combined human + *E. coli* FASTA database
- Installing Docker in WSL/Linux
- Building a DIA-NN Docker image
- Converting Thermo `.raw` files to `.mzML` with MSConvert
- Generating DIA-NN predicted spectral libraries from FASTA
- Processing DIA files with DIA-NN
- Comparing DIA and DDA results at protein and peptide level
- Reproducing publication-style plots for identification counts, overlap, data completeness, and reproducibility

---

## R analysis script

The downstream R script used for the DIA vs DDA comparison and publication-style figures is available here:

```text
https://sync.academiccloud.de/index.php/s/ZBfmW5ncuB69VAj
