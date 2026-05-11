# Spectral-library-based bottom-up proteomics analysis with DIA-NN

This repository contains a short tutorial for **spectral-library-based bottom-up proteomics analysis using DIA-NN**.

The tutorial uses the public PRIDE dataset **PXD062423**:

**Tear Fluid Proteomics: A Comparative Study of DIA and DDA Mass Spectrometry**

The workflow covers:

1. downloading DIA raw files and DIA-NN report tables from PRIDE;
2. downloading reviewed UniProt Swiss-Prot FASTA files;
3. creating a combined human + *E. coli* FASTA database;
4. installing Docker in WSL/Linux;
5. building a DIA-NN Docker image;
6. preparing the environment for DIA-NN analysis.

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
sudo apt install -y wget curl git gzip unzip ca-certificates gnupg lsb-release
```

---

# 0. Create the project folder

All data, FASTA files, Docker files, and DIA-NN results should be stored inside one project folder.

For this tutorial, use:

```bash
~/projects/diann_tutorial
```

Create the project folder:

```bash
mkdir -p ~/projects/diann_tutorial
cd ~/projects/diann_tutorial
```

From now on, all commands should be run from this project folder unless stated otherwise.

Check your current location:

```bash
pwd
```

Expected output should be similar to:

```text
/home/your_username/projects/diann_tutorial
```

Create the main folder structure:

```bash
mkdir -p data_prot/PXD062423
mkdir -p fasta
mkdir -p results/diann/libraries
mkdir -p tools/diann_docker
```

The project folder should now contain:

```bash
ls
```

Expected output:

```text
data_prot  fasta  results  tools
```
## Important: always work from the project root

All scripts in this tutorial assume that you run commands from the project root:

```bash
~/projects/diann_tutorial
```

# 1. Download the training dataset

The training dataset is downloaded from PRIDE project **PXD062423**.

This dataset contains Thermo `.raw` DIA files from a tear-fluid proteomics study, together with DIA-NN output tables.

Go to the dataset folder:

```bash
cd ~/projects/diann_tutorial/data_prot/PXD062423
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

Return to the project root:

```bash
cd ~/projects/diann_tutorial
```

---

# 2. Download FASTA protein databases

DIA-NN needs a **FASTA protein database** to generate a predicted spectral library and perform peptide identification.

For this tutorial, we use reviewed UniProt Swiss-Prot FASTA files:

- **Human Swiss-Prot FASTA** for the human tear-fluid proteome.
- **E. coli K-12 Swiss-Prot FASTA** because the dilution experiment contains human tear peptides mixed with *E. coli* digest.
- **Combined human + E. coli FASTA** so all files can be processed using one database.

Reviewed Swiss-Prot entries are preferred here because they are smaller, cleaner, and more suitable for a teaching workflow than the full UniProt database.

Go to the FASTA folder:

```bash
cd ~/projects/diann_tutorial/fasta
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

Return to the project root:

```bash
cd ~/projects/diann_tutorial
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

```bash
sudo apt update

sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
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

# 4. Build a DIA-NN Docker image

DIA-NN provides a Linux package that includes a Docker build script. This is the cleanest way to run DIA-NN in WSL/Linux because the container includes the required dependencies instead of relying on the host system.

Go to the DIA-NN Docker build folder:

```bash
cd ~/projects/diann_tutorial/tools/diann_docker
```

Download the public DIA-NN Academia Linux package from the official DIA-NN GitHub release:

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

Check the unpacked files:

```bash
ls -lh diann_linux
```

You should see files similar to:

```text
Dockerfile
make-docker.sh
diann-2.0
```

Go to the unpacked DIA-NN folder:

```bash
cd diann_linux
```

If the script has Windows line endings, fix them:

```bash
sed -i 's/\r$//' make-docker.sh
```

Build the DIA-NN Docker image:

```bash
chmod +x make-docker.sh
./make-docker.sh
```

Check that the image exists:

```bash
docker images | grep -i diann
```

Tag the built image as `diann:latest`:

```bash
docker tag diann_docker:latest diann:latest
```

Find the DIA-NN binary inside the image:

```bash
docker run --rm --entrypoint /bin/bash diann:latest -lc 'find / -type f -name "diann-linux" 2>/dev/null'
```

Create a wrapper image called `diann-run:latest`.

This wrapper lets us run DIA-NN directly without manually specifying the binary path every time:

```bash
DIANN_BIN=$(docker run --rm --entrypoint /bin/bash diann:latest -lc 'find / -type f -name "diann-linux" 2>/dev/null | head -n 1')

echo "$DIANN_BIN"

cat > Dockerfile.diann-run <<EOF
FROM diann:latest
ENTRYPOINT ["$DIANN_BIN"]
EOF

docker build -f Dockerfile.diann-run -t diann-run:latest .
```

Test the DIA-NN wrapper image:

```bash
docker run --rm diann-run:latest --help | head -n 30
```

Return to the project root:

```bash
cd ~/projects/diann_tutorial
```

---

# 5. Expected folder structure

After downloading the dataset, FASTA files, and DIA-NN Docker files, the project folder should look like this:

```text
~/projects/diann_tutorial
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
├── results
│   └── diann
│       └── libraries
└── tools
    └── diann_docker
        ├── diann_linux.zip
        └── diann_linux
```

---

# Notes

The `.raw` files are Thermo raw mass spectrometry files.

For DIA-NN analysis inside Docker, make sure the DIA-NN version supports Thermo `.raw` files on Linux. If not, convert the `.raw` files to `.mzML` before DIA-NN processing.

For this tutorial, the recommended FASTA file for all mixed human + *E. coli* dilution samples is:

```text
fasta/human_ecoli_swissprot_reviewed.fasta
```

For pooled human tear-fluid samples only, the human-only FASTA is also appropriate:

```text
fasta/human_swissprot_reviewed.fasta
```
DIA-NN supports only mzml files, but we have .raw ThermoScientific original files, so lets pull the MSConvert (PRoteowizard docker) and convert the files
```text
docker pull chambm/pwiz-skyline-i-agree-to-the-vendor-licenses
```
