# Spectral-library-based bottom-up proteomics with DIA-NN

A hands-on tutorial for spectral-library-based DIA proteomics using DIA-NN, built around the public PRIDE dataset **PXD062423** — a tear-fluid proteomics study comparing DIA and DDA mass spectrometry.

The workflow goes from raw file download through FASTA preparation, Docker setup, and DIA-NN analysis.

---

## What's covered

- Downloading DIA raw files and DIA-NN report tables from PRIDE
- Downloading reviewed UniProt Swiss-Prot FASTA files (human + *E. coli*)
- Building a combined FASTA database
- Installing Docker in WSL/Linux
- Building a DIA-NN Docker image
- Converting Thermo `.raw` files to `.mzML` with MSConvert

---

## Requirements

Linux or Windows with WSL Ubuntu. All commands run in a bash shell.

Install the basic tools you'll need:

```bash
sudo apt update
sudo apt install -y wget curl git gzip unzip ca-certificates gnupg lsb-release
```

---

## Project layout

Everything lives under one folder:

```bash
mkdir -p ~/projects/diann_tutorial
cd ~/projects/diann_tutorial
```

Create the folder structure before starting:

```bash
mkdir -p data_prot/PXD062423
mkdir -p fasta
mkdir -p results/diann/libraries
mkdir -p tools/diann_docker
```

> Run all commands from the project root unless stated otherwise.

---

## Download the dataset

The training data comes from PRIDE project **PXD062423** — Thermo `.raw` DIA files from a tear-fluid proteomics experiment plus DIA-NN output tables.

```bash
cd ~/projects/diann_tutorial/data_prot/PXD062423

BASE_URL="https://ftp.pride.ebi.ac.uk/pride/data/archive/2025/10/PXD062423"

for FILE in \
  DIA_Base_1.raw DIA_Base_2.raw DIA_Base_3.raw \
  DIA_2X_1.raw   DIA_2X_2.raw   DIA_2X_3.raw   \
  DIA_4X_1.raw   DIA_4X_2.raw   DIA_4X_3.raw   \
  DIA_8X_1.raw   DIA_8X_2.raw   DIA_8X_3.raw   \
  Pool_DIA_R1.raw Pool_DIA_R2.raw Pool_DIA_R3.raw Pool_DIA_R4.raw \
  Pool_DIA_R5.raw Pool_DIA_R6.raw Pool_DIA_R7.raw Pool_DIA_R8.raw \
  report_DIA.tsv report_spike.tsv
do
  wget -c "${BASE_URL}/${FILE}"
done

cd ~/projects/diann_tutorial
```

---

## Download FASTA databases

DIA-NN needs a FASTA database to generate a predicted spectral library and do peptide identification. We use reviewed Swiss-Prot entries — smaller, cleaner, and better suited to a teaching workflow than the full UniProt database.

The dilution experiment mixes human tear peptides with an *E. coli* digest, so we need both organisms and combine them into a single database.

```bash
cd ~/projects/diann_tutorial/fasta

# Human Swiss-Prot
wget -O human_swissprot_reviewed.fasta.gz \
  "https://rest.uniprot.org/uniprotkb/stream?compressed=true&format=fasta&query=%28reviewed%3Atrue%29%20AND%20%28organism_id%3A9606%29"
gunzip -kf human_swissprot_reviewed.fasta.gz

# E. coli K-12 Swiss-Prot
wget -O ecoli_swissprot_reviewed.fasta.gz \
  "https://rest.uniprot.org/uniprotkb/stream?compressed=true&format=fasta&query=%28reviewed%3Atrue%29%20AND%20%28organism_id%3A83333%29"
gunzip -kf ecoli_swissprot_reviewed.fasta.gz

# Combine into one file
cat human_swissprot_reviewed.fasta ecoli_swissprot_reviewed.fasta > human_ecoli_swissprot_reviewed.fasta
```

Check the entry counts:

```bash
grep -c "^>" human_swissprot_reviewed.fasta    # ~20 400
grep -c "^>" ecoli_swissprot_reviewed.fasta    # ~4 500
grep -c "^>" human_ecoli_swissprot_reviewed.fasta  # ~24 900
```

Exact numbers shift slightly as UniProt is updated.

```bash
cd ~/projects/diann_tutorial
```

---

## Install Docker in WSL/Linux

Docker lets us run DIA-NN in a reproducible container without worrying about host-level dependencies.

```bash
# Add Docker's GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add the Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Start Docker and verify
sudo service docker start
docker --version || sudo docker --version
sudo docker run hello-world

# Add your user to the docker group to drop sudo requirements
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
```

If you still get a permission error after `newgrp`, close WSL and reopen it.

---

## Build the DIA-NN Docker image

DIA-NN ships a Linux package with a Docker build script. We pull the latest 2.0 release, build the image, and wrap it so DIA-NN can be called directly without specifying the binary path each time.

```bash
cd ~/projects/diann_tutorial/tools/diann_docker

# Download the Linux release from GitHub
DIANN_URL=$(curl -sL https://api.github.com/repos/vdemichev/DiaNN/releases/tags/2.0 | \
  grep browser_download_url | grep -Ei "Linux.*zip" | head -n 1 | cut -d '"' -f 4)

curl -L -o diann_linux.zip "$DIANN_URL"
unzip -q diann_linux.zip -d diann_linux
cd diann_linux

# Fix Windows line endings if present
sed -i 's/\r$//' make-docker.sh

# Build the image
chmod +x make-docker.sh
./make-docker.sh

docker tag diann_docker:latest diann:latest
```

Create a thin wrapper image so DIA-NN can be called without spelling out the binary path:

```bash
DIANN_BIN=$(docker run --rm --entrypoint /bin/bash diann:latest \
  -lc 'find / -type f -name "diann-linux" 2>/dev/null | head -n 1')

cat > Dockerfile.diann-run <<EOF
FROM diann:latest
ENTRYPOINT ["$DIANN_BIN"]
EOF

docker build -f Dockerfile.diann-run -t diann-run:latest .

# Quick sanity check
docker run --rm diann-run:latest --help | head -n 30
```

```bash
cd ~/projects/diann_tutorial
```

---

## Convert .raw files to .mzML

DIA-NN on Linux requires `.mzML` input — it does not read Thermo `.raw` files directly. Use the ProteoWizard MSConvert Docker image for the conversion.

```bash
docker pull chambm/pwiz-skyline-i-agree-to-the-vendor-licenses
```

Convert a file:

```bash
docker run --rm \
  -v ~/projects/diann_tutorial/data_prot/PXD062423:/data \
  chambm/pwiz-skyline-i-agree-to-the-vendor-licenses \
  wine msconvert /data/Pool_DIA_R1.raw --mzML --outdir /data
```

Repeat for all `.raw` files or wrap in a loop.

---

## Expected folder structure

After completing the steps above the project root should look like this:

```text
~/projects/diann_tutorial
├── data_prot
│   └── PXD062423
│       ├── DIA_Base_1.raw  ...  DIA_8X_3.raw
│       ├── Pool_DIA_R1.raw  ...  Pool_DIA_R8.raw
│       ├── report_DIA.tsv
│       └── report_spike.tsv
├── fasta
│   ├── human_swissprot_reviewed.fasta
│   ├── ecoli_swissprot_reviewed.fasta
│   └── human_ecoli_swissprot_reviewed.fasta
├── results
│   └── diann
│       └── libraries
└── tools
    └── diann_docker
        ├── diann_linux.zip
        └── diann_linux
```

---

## Which FASTA to use

| Samples | Recommended FASTA |
|---|---|
| Mixed human + *E. coli* dilution series | `fasta/human_ecoli_swissprot_reviewed.fasta` |
| Pooled human tear fluid only | `fasta/human_swissprot_reviewed.fasta` |
