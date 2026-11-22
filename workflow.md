# Full workflow

> Scripts and files from other sources (CodonW, CAIcal, the Perl scripts, human reference files and
> Excel templates) are available **upon request**; see [`data/README.md`](../data/README.md).

Steps marked **[automated]** are implemented in this repository; steps marked **[manual]**
use external software and are documented here so the analysis can be reproduced end to end.

```
Step 1  Download CDS                [automated]  codon-usage download
Step 2  Preprocess                  [automated]  codon-usage filter / rename
Step 3  Analysis
  Part I   CodonW indices           [manual] + [automated parsing]
  Part II  CAI, RCDI                [manual] CAIcal
  Part III dinuc, tAI, RSCPU, SID,  [manual / Perl / R / Excel / SPSS]
           correlation
Step 4  Figures                     [automated] + Excel templates
Step 5  Tables                      [manual]
```

## Step 1 - Download coding sequences
Put GenBank accession IDs in `id.txt` (one per line), then:
```bash
codon-usage download id.txt --email you@example.com -o all_cds_sequences.fasta
```
Skip if you already have the sequences.

## Step 2 - Preprocessing
```bash
codon-usage filter all_cds_sequences.fasta -o filtered.fasta -m filter_log.csv
codon-usage rename filtered.fasta --prefix RVA -o final.fasta -i header_index.tsv
```
`filter` removes CDS with non-ACGT characters, length not divisible by 3, or internal stop codons,
and writes a kept/removed log. `rename` replaces headers with `<prefix>_1, <prefix>_2, ...`
and saves an original-to-new ID index.

## Step 3 - Analysis

Indices computed:
1. Codon usage indices and nucleotide composition (ENC, A/T/G/C, GC3, ...)
2. Relative synonymous codon usage (RSCU)
3. Relative amino acid usage (RAAU)
4. Overall RSCU
5. Codon adaptation index (CAI)
6. Relative codon deoptimization index (RCDI)
7. Dinucleotide frequency (observed/expected)
8. tRNA adaptation index (tAI)
9. Relative synonymous codon pair usage (RSCPU)
10. Similarity index (SID)
11. Correlation

### Part I - CodonW (manual)
Put the sequence file (`.dat` format) and `CodonW.exe` in the same folder.

**1A. Codon usage indices** - open CodonW; load sequence file (option 1); enter the file name; press
Enter for the output and bulk files; run codon usage indices (option 4); select all (option 12);
return (X); run C codons (R); quit. Save `sequence.out` to Excel.

**1B. Exhaustive base composition** - load sequence file (1); enter the file name; Enter for the
output and bulk files; change the output written to file (8); exhaustive base composition (9);
return (X); run (R); quit. Extract GC_not_GC3s and the W, S, R, Y (and C, T) totals per gene:
```bash
codon-usage parse-codonw sequence.out -o composition.xlsx
```
Merge with 1A and calculate average, count and standard deviation per column.

**2. RSCU** - as above but choose option 8 then RSCU (option 6). Save the `genes.coa` output.

**3. RAAU** - as above but option 8 then RAAU (option 5). Save `genes.coa`; compute average, count, SD.

**4. Overall RSCU** - from a command prompt in the folder run `codonw sequence.dat -cutot`, then use
option 8 and RSCU (6). Save the `sequence.bulk` table.

### Part II - CAI and RCDI (CAIcal)
Use the web server http://genomes.urv.es/CAIcal/ (CAI, RCDI, "Calculation of parameters") with the
human codon usage table and the standard genetic code, or run locally with Perl:
```bash
# CAI
perl CAIcal_ECAI_v1.4.pl -e cai_and_expected -f example.ffn -h human.txt -g 1 -p 95 \
  -o1 cai_and_expected.txt -n 1000 -l 100 -m markov -gc 50 -s y
# RCDI
perl CAIcal_ECAI_v1.4.pl -e rcdi_and_ercdi -f example.ffn -h human.txt -g 1 -p 95 \
  -o1 rcdi_and_ercdi.txt -n 1000 -l 100 -m markov -gc 50 -s y
```
Local files: `CAIcal_ECAI_v1.4.pl`, `example.ffn` (FASTA as .ffn), `human.txt`.

### Part III
**7. Dinucleotide frequency (obs/exp)** - `perl di_nt_freq.pl finalcds.fasta > dinuc.txt`

**8. tAI** - `perl codonM sequence.ffn virus.m`, then run `Rscript r/tai.R` (needs `human.trna`
and `virus.m`). Check that `tai.csv` has one row per sequence; report mean and SD.

**9. RSCPU / codon pair score** - run `perl CPS_20.pl` in a folder with the sequence FASTA (it
prompts for the file name) to get `cps-tab.csv`. Then:
```bash
codon-usage codon-pairs --cps cps-tab.csv --valid-pairs human_codon_pair.txt \
  --junctions junctions.txt --over human_over.txt --under human_under.txt
```
This writes filtered pairs, average CPS per junction, and the over-/under-represented pairs that
match human, with percentages.

**10. SID** - fill the SID Excel template with your virus RSCU values in the designated row;
D(a,b) is the SID value.

**11. Correlation** - collect parameters in Excel and run correlation (Pearson/Spearman) in SPSS.

## Step 4 - Figures
| Figure | How |
|---|---|
| Nucleotide composition | `codon-usage plot-composition params.csv` |
| GC3 vs ENC | Excel scatter (GC3 on x, ENC on y) with the expected-curve overlay from the template |
| GC3 vs GC12 (neutrality) | Excel scatter with trendline and equation |
| Relative dinucleotide abundance | Excel bar chart vs human (template) |
| Codon pair score | Excel bar chart of average CPS per junction vs human (template) |
| Correlation heatmap | `codon-usage plot-heatmap spss_table.csv --method pearson` (or `spearman`) |
| CAI boxplots with Wilcoxon p-values | `Rscript r/cai_boxplot.R` |

## Step 5 - Tables
- **RSCU table**: use overall RSCU (Step 3, Part I, #4). Highlight codons with RSCU > 1 (preferred codons)
  and highlight antagonistic/coincident codons (see the reference paper you follow).
- **tRNA table**: for each amino acid use the codon with the highest overall RSCU.
