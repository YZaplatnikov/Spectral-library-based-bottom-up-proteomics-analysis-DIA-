#!/usr/bin/env bash
set -euo pipefail

# =========================
# USER PATHS — EDIT THESE
# =========================
RAW_DIR="$HOME/data_prot/PXD062423_DDA"
MZML_DIR="$HOME/data_prot/PXD062423_DDA/mzML"
OUT_DIR="$HOME/data_prot/PXD062423_DDA/fragger_out"
FASTA="$HOME/data_prot/db/uniprot_sprot_human_2024.fasta"

MSCONVERT="/path/to/msconvert"                       # ProteoWizard msconvert
JAVA_BIN="java"
MSFRAGGER_JAR="$HOME/tools/MSFragger-4.1/MSFragger-4.1.jar"
PHILOSOPHER="$HOME/tools/philosopher/philosopher"
IONQUANT_JAR="$HOME/tools/IonQuant-1.10.27/IonQuant-1.10.27.jar"

THREADS=8

mkdir -p "$MZML_DIR" "$OUT_DIR"

# =========================
# 1. Convert Thermo RAW -> mzML
# =========================
echo "Converting RAW to mzML..."
for f in "$RAW_DIR"/Pool_DDA_R*.raw; do
  "$MSCONVERT" "$f" \
    --mzML \
    --filter "peakPicking true 1-" \
    --outfile "$(basename "${f%.raw}.mzML")" \
    -o "$MZML_DIR"
done

# =========================
# 2. Create MSFragger parameter file
# =========================
PARAMS_FILE="$OUT_DIR/fragger_dda.params"

cat > "$PARAMS_FILE" <<'EOF'
# =========================
# MSFragger params for pooled DDA tear-fluid files
# =========================

database_name = __FASTA__
num_threads = __THREADS__

precursor_mass_lower = -20
precursor_mass_upper = 20
precursor_mass_units = 1          # 1 = ppm
precursor_true_tolerance = 10
precursor_true_units = 1

fragment_mass_tolerance = 0.6
fragment_mass_units = 0           # 0 = Da

calibrate_mass = 1
use_topN_peaks = 150
minimum_peaks = 10

search_enzyme_name = Trypsin
search_enzyme_cutafter = KR
search_enzyme_butnotafter = P
num_enzyme_termini = 2
allowed_missed_cleavage = 1

digest_min_length = 7
digest_max_length = 50

precursor_charge = 1 7
override_charge = 0
digest_mass_range = 500 5000

variable_mod_01 = 15.9949 M 3
add_C_cysteine = 57.021464

clip_nTerm_M = 1

track_zero_topN = 0
zero_bin_accept_expect = 1
isotope_error = 0/1/2

output_format = pepXML
output_report_topN = 1
report_alternative_proteins = 0
EOF

# Replace placeholders
sed -i "s#__FASTA__#$FASTA#g" "$PARAMS_FILE"
sed -i "s#__THREADS__#$THREADS#g" "$PARAMS_FILE"

# =========================
# 3. Run MSFragger search
# =========================
echo "Running MSFragger..."
"$JAVA_BIN" -Xmx32G -jar "$MSFRAGGER_JAR" "$PARAMS_FILE" "$MZML_DIR"/Pool_DDA_R*.mzML

# Move pepXML outputs into OUT_DIR
find "$MZML_DIR" -maxdepth 1 -name "Pool_DDA_R*.pepXML" -exec mv {} "$OUT_DIR" \;

cd "$OUT_DIR"

# =========================
# 4. Philosopher workspace + FASTA annotation
# =========================
echo "Running Philosopher..."
"$PHILOSOPHER" workspace --init
"$PHILOSOPHER" database --annotate "$FASTA"

# PeptideProphet / iProphet / ProteinProphet-style validation
"$PHILOSOPHER" peptideprophet --decoy rev_ --ppm --accmass --nonparam --expectscore Pool_DDA_R*.pepXML
"$PHILOSOPHER" proteinprophet interact-*.pep.xml

# Filter at 1% peptide and protein FDR
"$PHILOSOPHER" filter --psm 0.01 --pep 0.01 --prot 0.01

# Report tables
"$PHILOSOPHER" report

# =========================
# 5. IonQuant label-free quantification
# =========================
echo "Running IonQuant..."
"$JAVA_BIN" -Xmx24G -jar "$IONQUANT_JAR" \
  --threads "$THREADS" \
  --tol 10 \
  --ptol 10 \
  --mztol 10 \
  --files "$MZML_DIR"/Pool_DDA_R*.mzML \
  --psm "$OUT_DIR/psm.tsv"

echo "Done."
echo "Main outputs should be in: $OUT_DIR"
