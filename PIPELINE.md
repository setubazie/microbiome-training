# 16S rRNA Amplicon Sequencing Pipeline — Step-by-Step

This walks through everything needed to go from the raw FASTQ files in `data/` to taxonomy and diversity results, using QIIME 2. It covers installation on Mac and Windows (via WSL), then the exact commands to run on this demo dataset (6 samples, Dairy vs. Meat, 3 replicates each).

---

## 1. Before you start

You'll need:
- A Mac, or a Windows PC with WSL (Windows Subsystem for Linux) enabled
- About 10 GB free disk space
- The 12 `.fastq.gz` files from this repo's `data/` folder, downloaded to your computer (unlock them on the [training page](https://setubazie.github.io/microbiome-training/) with the password you were given)

---

## 2. Install Miniconda

QIIME 2 is installed through conda, so Miniconda (a lightweight conda installer) comes first.

### macOS

1. Open **Terminal** (Spotlight search → "Terminal").
2. Download and run the installer:
   ```bash
   curl -L -O "https://repo.anaconda.com/miniconda/Miniconda3-latest-MacOSX-$(uname -m).sh"
   bash Miniconda3-latest-MacOSX-$(uname -m).sh
   ```
3. Accept the license, accept the default install location, and say **yes** when asked whether to run `conda init`.
4. Close and reopen Terminal so the changes take effect.
5. Confirm it worked:
   ```bash
   conda --version
   ```

### Windows (via WSL)

QIIME 2 does not run natively on Windows — you need WSL (a real Linux environment inside Windows).

1. Open **PowerShell as Administrator** (right-click Start → "Windows Terminal (Admin)" or "PowerShell (Admin)").
2. Install WSL with Ubuntu:
   ```powershell
   wsl --install
   ```
3. Restart your computer when prompted.
4. After restart, a Ubuntu terminal window should open automatically (or search "Ubuntu" in the Start menu). Set a username and password when asked — this is separate from your Windows login.
5. Inside that Ubuntu terminal, download and install Miniconda:
   ```bash
   wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
   bash Miniconda3-latest-Linux-x86_64.sh
   ```
6. Accept the license, accept the default location, and say **yes** to `conda init`.
7. Close and reopen the Ubuntu terminal.
8. Confirm it worked:
   ```bash
   conda --version
   ```

> From here on, every command in this guide is run inside **Terminal** (Mac) or the **Ubuntu/WSL terminal** (Windows) — not PowerShell, and not Windows' regular Command Prompt.

---

## 3. Install QIIME 2

As of this training, QIIME 2 is installed from a `.yml` environment file (check [qiime2.org/install](https://docs.qiime2.org/) for the current version number if this one is outdated by the time you set up):

```bash
conda update conda -y
conda install -n base conda-libmamba-solver -y
conda config --set solver libmamba

# Mac (Apple Silicon or Intel):
curl -L -O https://data.qiime2.org/distro/amplicon/qiime2-amplicon-2024.10-py310-osx-conda.yml
conda env create -n qiime2-amplicon-2024.10 --file qiime2-amplicon-2024.10-py310-osx-conda.yml

# Windows/WSL (Linux):
wget https://data.qiime2.org/distro/amplicon/qiime2-amplicon-2024.10-py310-linux-conda.yml
conda env create -n qiime2-amplicon-2024.10 --file qiime2-amplicon-2024.10-py310-linux-conda.yml
```

This takes a while (10–20 minutes) — it's installing a large scientific software stack.

Activate the environment (do this every time you open a new terminal to work on this):

```bash
conda activate qiime2-amplicon-2024.10
```

Confirm it worked:

```bash
qiime --help
```

You should see a list of QIIME 2 commands, not an error.

---

## 4. Set up your working folder

```bash
mkdir ~/microbiome-training && cd ~/microbiome-training
mkdir data
```

Copy the 12 downloaded `.fastq.gz` files into this `data/` folder (drag-and-drop works fine on Mac Finder; on Windows/WSL, your Windows Downloads folder is reachable at `/mnt/c/Users/<yourname>/Downloads/`).

---

## 5. Create the sample metadata file

Create a file named `sample-metadata.tsv` in `~/microbiome-training/` with this content (tab-separated — if you build it in Excel/Sheets, save as "Tab-delimited text"):

| sample-id | Treatment | Replication |
|---|---|---|
| 1_S146_10 | Dairy | 1 |
| 3_S148_29 | Dairy | 2 |
| 5_S150_30 | Dairy | 3 |
| 11_S156_1 | Meat | 1 |
| 12_S157_2 | Meat | 2 |
| 14_S159_4 | Meat | 3 |

---

## 6. Create the manifest file and import into QIIME 2

Create `manifest.csv` in `~/microbiome-training/` listing each sample's forward/reverse file paths:

```csv
sample-id,forward-absolute-filepath,reverse-absolute-filepath
1_S146_10,$PWD/data/1_S146_R1_001.fastq.gz,$PWD/data/1_S146_R2_001.fastq.gz
3_S148_29,$PWD/data/3_S148_R1_001.fastq.gz,$PWD/data/3_S148_R2_001.fastq.gz
5_S150_30,$PWD/data/5_S150_R1_001.fastq.gz,$PWD/data/5_S150_R2_001.fastq.gz
11_S156_1,$PWD/data/11_S156_R1_001.fastq.gz,$PWD/data/11_S156_R2_001.fastq.gz
12_S157_2,$PWD/data/12_S157_R1_001.fastq.gz,$PWD/data/12_S157_R2_001.fastq.gz
14_S159_4,$PWD/data/14_S159_R1_001.fastq.gz,$PWD/data/14_S159_R2_001.fastq.gz
```

> `$PWD` only expands correctly if you run the import command from inside `~/microbiome-training/` — if it doesn't work, replace `$PWD` with the full absolute path yourself (e.g. `/Users/yourname/microbiome-training` or `/home/yourname/microbiome-training`).

Import:

```bash
qiime tools import \
  --type 'SampleData[PairedEndSequencesWithQuality]' \
  --input-path manifest.csv \
  --input-format PairedEndFastqManifestPhred33V2 \
  --output-path demux.qza
```

Summarize so you can inspect read quality:

```bash
qiime demux summarize \
  --i-data demux.qza \
  --o-visualization demux.qzv
```

View it at [view.qiime2.org](https://view.qiime2.org) by uploading `demux.qzv` — this tells you where read quality drops off, which you'll need for the next step.

---

## 7. Denoise with DADA2

Trim/truncate lengths depend on what you saw in `demux.qzv`. As a starting point for 2×250bp MiSeq-style reads (adjust based on your actual quality plot):

```bash
qiime dada2 denoise-paired \
  --i-demultiplexed-seqs demux.qza \
  --p-trim-left-f 0 \
  --p-trim-left-r 0 \
  --p-trunc-len-f 230 \
  --p-trunc-len-r 200 \
  --o-representative-sequences rep-seqs.qza \
  --o-table table.qza \
  --o-denoising-stats denoising-stats.qza \
  --p-n-threads 0
```

Summarize the results:

```bash
qiime feature-table summarize \
  --i-table table.qza \
  --o-visualization table.qzv \
  --m-sample-metadata-file sample-metadata.tsv

qiime feature-table tabulate-seqs \
  --i-data rep-seqs.qza \
  --o-visualization rep-seqs.qzv
```

---

## 8. Assign taxonomy

Download a pre-trained classifier (SILVA 138, 99% OTUs, full-length 16S — ~140MB):

```bash
wget https://data.qiime2.org/2024.10/common/silva-138-99-nb-classifier.qza
```

Classify:

```bash
qiime feature-classifier classify-sklearn \
  --i-classifier silva-138-99-nb-classifier.qza \
  --i-reads rep-seqs.qza \
  --o-classification taxonomy.qza

qiime metadata tabulate \
  --m-input-file taxonomy.qza \
  --o-visualization taxonomy.qzv
```

Taxa bar plot:

```bash
qiime taxa barplot \
  --i-table table.qza \
  --i-taxonomy taxonomy.qza \
  --m-metadata-file sample-metadata.tsv \
  --o-visualization taxa-bar-plots.qzv
```

---

## 9. Build a phylogenetic tree

```bash
qiime phylogeny align-to-tree-mafft-fasttree \
  --i-sequences rep-seqs.qza \
  --o-alignment aligned-rep-seqs.qza \
  --o-masked-alignment masked-aligned-rep-seqs.qza \
  --o-tree unrooted-tree.qza \
  --o-rooted-tree rooted-tree.qza
```

---

## 10. Alpha and beta diversity

Pick a sampling depth from `table.qzv` (choose a value that keeps as many samples as possible without excluding low-depth ones — for this small demo, check the per-sample feature counts first).

```bash
qiime diversity core-metrics-phylogenetic \
  --i-phylogeny rooted-tree.qza \
  --i-table table.qza \
  --p-sampling-depth 1000 \
  --m-metadata-file sample-metadata.tsv \
  --output-dir core-metrics-results
```

(Replace `1000` with your chosen depth.)

### Alpha diversity — Kruskal-Wallis (comparing Dairy vs. Meat)

```bash
qiime diversity alpha-group-significance \
  --i-alpha-diversity core-metrics-results/shannon_vector.qza \
  --m-metadata-file sample-metadata.tsv \
  --o-visualization core-metrics-results/shannon-group-significance.qzv
```

### Beta diversity — PERMANOVA

```bash
qiime diversity beta-group-significance \
  --i-distance-matrix core-metrics-results/weighted_unifrac_distance_matrix.qza \
  --m-metadata-file sample-metadata.tsv \
  --m-metadata-column Treatment \
  --o-visualization core-metrics-results/weighted-unifrac-Treatment-significance.qzv \
  --p-pairwise
```

View every `.qzv` file produced above at [view.qiime2.org](https://view.qiime2.org).

---

## Troubleshooting

- **`conda: command not found` after installing** — close and fully reopen your terminal window; conda edits your shell's startup file, which only applies to new terminal sessions.
- **WSL "class not registered" or install fails on Windows** — make sure virtualization is enabled in your PC's BIOS, and that Windows is updated (WSL2 requires a reasonably recent Windows 10/11 build).
- **`qiime: command not found`** — you forgot to run `conda activate qiime2-amplicon-2024.10` in this terminal session; it needs to be run every time you open a new terminal window.
- **DADA2 step is very slow** — this is normal on a laptop for the first run; it can take 10–30+ minutes even for 6 samples.
