# Spectral-library-based bottom-up proteomics analysis with DIA-NN

This repository contains a short tutorial for **spectral-library-based bottom-up proteomics analysis using DIA-NN**.

The tutorial uses the public PRIDE dataset **PXD062423**:

**Tear Fluid Proteomics: A Comparative Study of DIA and DDA Mass Spectrometry**

The workflow covers:

1. downloading DIA raw files and DIA-NN report tables from PRIDE;
2. downloading reviewed UniProt Swiss-Prot FASTA files;
3. creating a combined human + *E. coli* FASTA database;
4. installing Docker in WSL/Linux;
5. preparing the environment for DIA-NN analysis.

---

## Requirements

Use either:

- Linux, or
- Windows with WSL Ubuntu.

All commands below should be run in a **bash shell**.

Do **not** run these commands in Windows PowerShell.

Install basic command-line tools:

```bash
sudo apt update
sudo apt install -y wget curl git gzip ca-certificates gnupg lsb-release
```

---

# 1. Download the training dataset

The training dataset is downloaded from PRIDE project **PXD062423**.

This dataset contains Thermo `.raw` DIA files from a tear-fluid proteomics study, together with DIA-NN output tables.

Create a folder for the dataset:

```bash
mkdir -p data_prot/PXD062423
cd data_prot/PXD062423
```

Define the PRIDE download link:

```bash
BASE_URL="https://ftp.pride.ebi.ac.uk/pride/data/archive/2025/10/PXD062423"
```

Download all raw files and DIA-NN report tables:

```bash
for FILE in \
DIA_Base_1.raw \
DIA_Base_2.raw \
DIA_Base_3.raw \
DIA_2X_1.raw \
DIA_2X_2.raw \
DIA_2X_3.raw \
DIA_4X_1.raw \
DIA_4X_2.raw \
DIA_4X_3.raw \
DIA_8X_1.raw \
DIA_8X_2.raw \
DIA_8X_3.raw \
Pool_DIA_R1.raw \
Pool_DIA_R2.raw \
Pool_DIA_R3.raw \
Pool_DIA_R4.raw \
Pool_DIA_R5.raw \
Pool_DIA_R6.raw \
Pool_DIA_R7.raw \
Pool_DIA_R8.raw \
report_DIA.tsv \
report_spike.tsv
do
  wget -c "${BASE_URL}/${FILE}"
done
```

Check the downloaded files:

```bash
ls -lh
```

Return to the repository root:

```bash
cd ../..
```

---

# 2. Download FASTA protein databases

DIA-NN needs a **FASTA protein database** to generate a predicted spectral library and perform peptide identification.

For this tutorial, we use reviewed UniProt Swiss-Prot FASTA files:

- **Human Swiss-Prot FASTA** for the human tear-fluid proteome.
- **E. coli K-12 Swiss-Prot FASTA** because the dilution experiment contains human tear peptides mixed with *E. coli* digest.
- **Combined human + E. coli FASTA** so all files can be processed using one database.

Reviewed Swiss-Prot entries are preferred here because they are smaller, cleaner, and more suitable for a teaching workflow than the full UniProt database.

Create a FASTA folder:

```bash
mkdir -p fasta
cd fasta
```

Download reviewed human Swiss-Prot FASTA:

```bash
wget -O human_swissprot_reviewed.fasta.gz "https://rest.uniprot.org/uniprotkb/stream?compressed=true&format=fasta&query=%28reviewed%3Atrue%29%20AND%20%28organism_id%3A9606%29"
```

Unzip the human FASTA file:

```bash
gunzip -kf human_swissprot_reviewed.fasta.gz
```

Download reviewed *E. coli* K-12 Swiss-Prot FASTA:

```bash
wget -O ecoli_swissprot_reviewed.fasta.gz "https://rest.uniprot.org/uniprotkb/stream?compressed=true&format=fasta&query=%28reviewed%3Atrue%29%20AND%20%28organism_id%3A83333%29"
```

Unzip the *E. coli* FASTA file:

```bash
gunzip -kf ecoli_swissprot_reviewed.fasta.gz
```

Create the combined human + *E. coli* FASTA:

```bash
cat human_swissprot_reviewed.fasta ecoli_swissprot_reviewed.fasta > human_ecoli_swissprot_reviewed.fasta
```

Check the number of protein entries:

```bash
grep -c "^>" human_swissprot_reviewed.fasta
grep -c "^>" ecoli_swissprot_reviewed.fasta
grep -c "^>" human_ecoli_swissprot_reviewed.fasta
```

Expected values are approximately:

```text
20400 human proteins
4500 E. coli proteins
24900 combined proteins
```

The exact numbers can differ slightly because UniProt is updated over time.

Return to the repository root:

```bash
cd ..
```

---

# 3. Install Docker in WSL/Linux

DIA-NN can be run inside a Docker container. Docker makes the analysis environment more reproducible because the same software setup can be used across different computers.

The commands below install Docker Engine inside WSL Ubuntu/Linux.

Add Docker’s official GPG key:

```bash
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

Add the Docker repository:

```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install Docker Engine:
(make sure you are in the folder "diann_linux"!)
```bash
sudo apt update

sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

If you have problem with WSL (Bash is trying to run /bin/bash\r, which does not exist), try this:
```bash
sed -i 's/\r$//' make-docker.sh
```
Start Docker:

```bash
sudo service docker start
```

Check Docker version:

```bash
docker --version || sudo docker --version
```

Test Docker:

```bash
sudo docker run hello-world
```

Add your user to the Docker group so Docker can be run without `sudo`:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Test Docker without `sudo`:

```bash
docker run hello-world
```

If Docker still gives a permission error, close WSL, open it again, and run:

```bash
docker run hello-world
```

---

#  Expected folder structure

After downloading the dataset and FASTA files, the repository should look like this:

```text
.
├── data_prot
│   └── PXD062423
│       ├── DIA_Base_1.raw
│       ├── DIA_Base_2.raw
│       ├── DIA_Base_3.raw
│       ├── DIA_2X_1.raw
│       ├── DIA_2X_2.raw
│       ├── DIA_2X_3.raw
│       ├── DIA_4X_1.raw
│       ├── DIA_4X_2.raw
│       ├── DIA_4X_3.raw
│       ├── DIA_8X_1.raw
│       ├── DIA_8X_2.raw
│       ├── DIA_8X_3.raw
│       ├── Pool_DIA_R1.raw
│       ├── Pool_DIA_R2.raw
│       ├── Pool_DIA_R3.raw
│       ├── Pool_DIA_R4.raw
│       ├── Pool_DIA_R5.raw
│       ├── Pool_DIA_R6.raw
│       ├── Pool_DIA_R7.raw
│       ├── Pool_DIA_R8.raw
│       ├── report_DIA.tsv
│       └── report_spike.tsv
├── fasta
│   ├── human_swissprot_reviewed.fasta
│   ├── ecoli_swissprot_reviewed.fasta
│   └── human_ecoli_swissprot_reviewed.fasta
└── README.md
```

---
# Notes

The `.raw` files are Thermo raw mass spectrometry files.

For DIA-NN analysis inside Docker, make sure the DIA-NN version supports Thermo `.raw` files on Linux. If not, convert the `.raw` files to `.mzML` before DIA-NN processing.

For this tutorial, the recommended FASTA file is:

```text
fasta/human_ecoli_swissprot_reviewed.fasta
```
# 4. Build a DIA-NN Docker image

DIA-NN provides a Linux package that includes a `make_docker.sh` script for building a Docker container. This is the cleanest way to run DIA-NN in WSL/Linux because the container includes the required dependencies instead of relying on the host system. The standard DIA-NN workflow is: first generate a predicted spectral library from FASTA, then analyze the raw files with that library. :contentReference[oaicite:0]{index=0}

Create a folder for the DIA-NN Docker build:

```bash
mkdir -p tools/diann_docker
cd tools/diann_docker
```

Download the latest public DIA-NN Academia Linux package from the official DIA-NN GitHub release:

```bash
DIANN_URL=$(curl -sL https://api.github.com/repos/vdemichev/DiaNN/releases/tags/2.0 | \
grep browser_download_url | \
grep -Ei "Linux.*zip" | \
head -n 1 | \
cut -d '"' -f 4)

echo "$DIANN_URL"

curl -L -o diann_linux.zip "$DIANN_URL"
```

Unzip the DIA-NN package:

```bash
unzip -q diann_linux.zip -d diann_linux
```

Find the Docker build script:

```bash
find diann_linux -name "make_docker.sh"
```

Enter the folder containing `make_docker.sh`:

```bash
cd $(find diann_linux -name "make_docker.sh" -exec dirname {} \; | head -n 1)
```

Build the DIA-NN Docker image:

```bash
chmod +x make_docker.sh
./make_docker.sh
```

Tag the newest image as `diann:latest` so that the commands below work consistently:

```bash
BUILT_IMAGE_ID=$(docker images -q | head -n 1)
docker tag "$BUILT_IMAGE_ID" diann:latest
```

Check that the image exists:

```bash
docker images | grep -i diann
```

Test DIA-NN inside Docker:

```bash
docker run --rm --entrypoint /bin/bash diann:latest -lc 'find / -type f -name "diann-linux" 2>/dev/null'

docker run --rm --entrypoint /diann-2.0/diann-linux diann:latest --help | head -n 30
```

Return to the repository root:

```bash
cd ../../../..
```

---
