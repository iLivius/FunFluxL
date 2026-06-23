# FunFluxL
Integrated workflow for fungal long-read genome assembly and annotation.

[![Snakemake](https://img.shields.io/badge/Snakemake-9.14.6%2B-brightgreen.svg)](https://snakemake.readthedocs.io/en/stable/)
[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.20796645.svg)](https://doi.org/10.5281/zenodo.20796645)

---
```bash
__________             _______________             ______
___  ____/___  ___________  ____/__  /___  _____  ____  /
__  /_   _  / / /_  __ \_  /_   __  /_  / / /_  |/_/_  /
_  __/   / /_/ /_  / / /  __/   _  / / /_/ /__>  < _  /___
/_/      \__,_/ /_/ /_//_/      /_/  \__,_/ /_/|_| /_____/

FunFluxL v0.1.0

June 2026
```
---

## Synopsis
`FunFluxL` is a [Snakemake](https://snakemake.readthedocs.io/en/stable/index.html) workflow designed for fungal genome assembly and annotation from long-read sequencing data. It supports both Oxford Nanopore (ONT) and PacBio HiFi reads, with technology-aware assembly and polishing choices for their different error profiles and data characteristics.

The downstream fungal annotation strategy follows the parental [FunFlux](https://github.com/iLivius/FunFlux) workflow: contig decontamination, `BUSCO` completeness assessment, ITS extraction and taxonomic assignment, repeat masking, `funannotate` prediction, `InterProScan`, `EggNOG-mapper`, `antiSMASH`, and final `funannotate` annotation.

Use `FunFlux` for paired-end Illumina short reads. Use `Funnotator`, bundled with `FunFlux`, for pre-assembled fungal genomes.

## Table of Contents
- [Rationale](#rationale)
- [Description](#description)
- [Installation](#installation)
- [Configuration](#configuration)
- [Running FunFluxL](#running-funfluxl)
- [Output](#output)
- [Acknowledgements](#acknowledgements)
- [Citation](#citation)
- [References](#references)

## Rationale
Long-read fungal genome sequencing can resolve repeats and structural complexity that are difficult for short-read assemblers. At the same time, long-read data require technology-specific decisions.

`FunFluxL` keeps the [FunFlux](https://github.com/iLivius/FunFlux) annotation logic but replaces short-read preprocessing and `SPAdes` assembly with long-read filtering, read QC, `Flye` assembly, long-read mapping, optional `Medaka` polishing for ONT, and PacBio HiFi-aware assembly mode.

## Description
Here's a breakdown of the `FunFluxL` workflow:

01. **Preprocessing and read QC:**
    * Long reads are filtered with [Filtlong](https://github.com/rrwick/Filtlong).
    * Raw and filtered reads are summarized with [NanoPlot](https://github.com/wdecoster/NanoPlot).

02. **Assembly:**
    * Filtered reads are assembled with [Flye](https://github.com/mikolmogorov/Flye).
    * Files ending in `_ont` use ONT Flye modes.
    * Files ending in `_pacbio` use Flye `--pacbio-hifi`.

03. **QC, Decontamination, Completeness Assessment, and ITS extraction:**
    * Filtered long reads are mapped back to contigs with [minimap2](https://github.com/lh3/minimap2) and [samtools](https://github.com/samtools/samtools).
    * ONT reads use minimap2 `map-ont`.
    * PacBio HiFi reads use minimap2 `map-hifi`.
    * Mapping output is evaluated with [QualiMap](http://qualimap.conesalab.org/).
    * Contigs are aligned against the [NCBI core nt](https://ftp.ncbi.nlm.nih.gov/blast/db/) database using [BLAST+](https://blast.ncbi.nlm.nih.gov/doc/blast-help/).
    * Contaminant contigs are evaluated with [BlobTools](https://github.com/DRL/blobtools) and selected with the shared [FunFlux](https://github.com/iLivius/FunFlux) taxonomy selector.
    * ONT assemblies can be polished with [Medaka](https://github.com/nanoporetech/medaka).
    * PacBio HiFi assemblies skip Medaka regardless of `medaka_model`.
    * Genome completeness is assessed with [BUSCO](https://busco.ezlab.org/) using `--auto-lineage-euk`.
    * ITS markers are detected and extracted with [ITSx](https://microbiology.se/software/itsx/).
    * ITS taxonomic assignment is performed with [SINTAX](https://www.drive5.com/sintax/) as implemented in [VSEARCH](https://github.com/torognes/vsearch) using the automatically downloaded [UNITE](https://unite.ut.ee/repository.php) SINTAX reference.

04. **Gene Prediction:**

    `FunFluxL` uses the same funannotate-based prediction strategy as [FunFlux](https://github.com/iLivius/FunFlux), including repeat masking before gene prediction. The main difference is the contig source passed into this shared annotation logic. The general flow is:

    ```text
    funannotate_preprocess -> repeat_masking -> funannotate_prediction
    ```

    The final contig source depends on the sample:

    - ONT with Medaka enabled: `03.post-processing/consensus/<sample>/consensus.fasta`.
    - ONT with Medaka disabled: `03.post-processing/contaminants/<sample>/assembly_decontam.fasta`.
    - PacBio HiFi: `03.post-processing/contaminants/<sample>/assembly_decontam.fasta`.

    Repeat masking can use:

    - direct [tantan](https://gitlab.com/mcfrith/tantan) softmasking;
    - a more advanced [RepeatModeler](https://github.com/Dfam-consortium/RepeatModeler) and [RepeatMasker](https://www.repeatmasker.org/RepeatMasker/) strategy, where repeats are first modeled from each assembly and then used to softmask the corresponding contigs.

05. **Gene Annotation:**

    The annotation stage follows FunFlux:

    - [InterProScan](https://github.com/ebi-pf-team/interproscan) is expected as an external local installation.
    - [EggNOG-mapper](https://github.com/eggnogdb/eggnog-mapper) is used for orthology and functional annotation.
    - [antiSMASH](https://github.com/antismash/antismash) detects secondary metabolite biosynthetic gene clusters.
    - Final annotation is performed with [funannotate](https://github.com/nextgenusfs/funannotate).

06. **Report:**
    * Results are parsed and aggregated with [MultiQC](https://github.com/MultiQC/MultiQC).

[⬆ Back to Table of Contents](#table-of-contents)

## Installation
`FunFluxL` uses the same broad installation model as FunFlux. Conda-managed tools are installed by Snakemake; large databases and licensed tools must be prepared manually.

1. **Download FunFluxL:**

    ```bash
    git clone https://github.com/iLivius/FunFluxL.git
    cd FunFluxL
    ```

2. **Install Snakemake:**

    ```bash
    conda create -c conda-forge -c bioconda -n snakemake snakemake
    conda activate snakemake
    ```

3. **Databases and external software:**

    * `NCBI core nt` database:

        Required for BLAST/BlobTools decontamination unless decontamination has already been handled upstream.

        ```bash
        rsync --list-only rsync://ftp.ncbi.nlm.nih.gov/blast/db/core_nt.*.gz | grep '.tar.gz' | awk '{print "ftp.ncbi.nlm.nih.gov/blast/db/" $NF}' > nt_links.list
        cat nt_links.list | parallel -j4 'rsync -h --progress rsync://{} .'
        find . -name '*.gz' | parallel -j4 'echo {}; tar -zxf {}'

        wget -c 'ftp://ftp.ncbi.nlm.nih.gov/pub/taxonomy/taxdump.tar.gz'
        tar -zxvf taxdump.tar.gz

        wget 'ftp://ftp.ncbi.nlm.nih.gov/blast/db/taxdb.tar.gz'
        tar -zxvf taxdb.tar.gz

        wget -c 'ftp://ftp.ncbi.nlm.nih.gov/pub/taxonomy/accession2taxid/nucl_gb.accession2taxid.gz'
        gunzip nucl_gb.accession2taxid.gz
        ```

        *NOTE: the complete NCBI core nt database and taxonomy-related files require more than 200 GB of disk space.*

    * `UNITE` database:

        Manual download is not required for the standard workflow. The config contains:

        ```yaml
        links:
          unite_its_link: https://s3.hpc.ut.ee/plutof-public/original/338a1413-6039-4e00-b5cf-410346a1e366.gz
        ```

        The workflow downloads and decompresses this file automatically into:

        ```text
        03.post-processing/ITS_extraction/unite_its_sintax.fasta
        ```

    * `eggNOG diamond` database:

        ```bash
        conda create -n eggnog-mapper eggnog-mapper=2.1.13
        conda activate eggnog-mapper
        mkdir /data/eggnog_db
        download_eggnog_data.py --data_dir /data/eggnog_db -y
        ```

        *NOTE: the eggNOG database requires roughly 50 GB of disk space.*

    * `GeneMark-ES/ET`:

        Download from the [GeneMark](http://topaz.gatech.edu/GeneMark/license_download.cgi) page and set `directories.genemark_dir` to the `gmes_linux_64_4` directory. If the GeneMark Perl scripts point to a fixed Perl path that is not valid in your environment, adjust their shebangs:

        ```bash
        cd /path/to/gmes_linux_64_4
        find . -type f -name "*.pl" -exec sed -i '1s|^#!/usr/bin/perl|#!/usr/bin/env perl|' {} +
        ./gmes_petap.pl
        ```

    * `InterProScan`:

        The version tested was v5.77-108.0, which is not downloaded by the workflow. Download the distribution from the official [InterProScan download instructions](https://interproscan-docs.readthedocs.io/en/v5/HowToDownload.html) or directly from the EBI FTP archive:

        - [interproscan-5.77-108.0-64-bit.tar.gz](https://ftp.ebi.ac.uk/pub/software/unix/iprscan/5/5.77-108.0/interproscan-5.77-108.0-64-bit.tar.gz)
        - [interproscan-5.77-108.0-64-bit.tar.gz.md5](https://ftp.ebi.ac.uk/pub/software/unix/iprscan/5/5.77-108.0/interproscan-5.77-108.0-64-bit.tar.gz.md5)

        Then initialize it and set `files.iprscan` to the `interproscan.sh` path:

        ```bash
        tar -pxvzf interproscan-5.77-108.0-64-bit.tar.gz
        cd interproscan-5.77-108.0
        python3 setup.py -f interproscan.properties
        ./interproscan.sh
        ```

    * Repeat masking tools:

        `tantan`, `RepeatModeler`, and `RepeatMasker` are installed through the workflow Conda environment. The advanced masking mode uses the custom RepeatModeler library with `RepeatMasker -lib`, so no separate curated RepeatMasker database is required for that mode.

    * Long-read tools:

        [Filtlong](https://github.com/rrwick/Filtlong), [NanoPlot](https://github.com/wdecoster/NanoPlot), [Flye](https://github.com/mikolmogorov/Flye), [minimap2](https://github.com/lh3/minimap2), and [Medaka](https://github.com/nanoporetech/medaka) are installed through workflow Conda environments.

[⬆ Back to Table of Contents](#table-of-contents)

## Configuration
Before running `FunFluxL`, edit `config/config.yaml`. The file is organized into `links`, `directories`, `files`, `resources`, and `parameters`.

- `links`

    - [funannotate_link](https://zenodo.org/records/18271295/files/funannotate_db_1.8.15_2025-12-15.tar.gz?download=1): URL to a frozen funannotate database snapshot. This is a database archive, not the funannotate executable version.
    - [unite_its_link](https://s3.hpc.ut.ee/plutof-public/original/338a1413-6039-4e00-b5cf-410346a1e366.gz): URL to the compressed UNITE SINTAX FASTA.

- `directories`

    - **input_dir**: Directory containing long-read FASTQ files.

        Expected naming:

        ```text
        isolateA_ont.fastq.gz
        isolateB_pacbio.fastq.gz
        ```

        Requirements:

        1. Use `_ont` for Oxford Nanopore reads.
        2. Use `_pacbio` for PacBio HiFi reads.
        3. ONT and PacBio HiFi samples can be analyzed in the same run, as long as each sample is provided with only one technology suffix.
        4. Files can only have `fastq`, `fq`, `fastq.gz`, or `fq.gz` extensions.
        5. All files in a run must use the same extension.
        6. Sample names must not contain underscores or these characters: `_*#@%^/! ?&:;|<>`.
        7. Do not provide both `sample_ont.*` and `sample_pacbio.*` for the same sample.

    - **output_dir**: Directory where output files, `.snakemake` metadata, and Conda environments are stored. Reusing the same output directory avoids reinstalling environments.
    - **blast_db**: Path to the `NCBI core nt` database and taxonomy files.
    - **eggnog_db**: Path to the `eggNOG-mapper` database.
    - **genemark_dir**: Path to the `gmes_linux_64_4` directory.
    - **funannotate_db**: Path where the `funannotate` database snapshot is installed or already available.

- `files`

    - **annotation_params**: Path to a tab-delimited annotation parameter file.

        | #Sample  | Species               | Proteins              | Model                |
        |----------|-----------------------|-----------------------|----------------------|
        | isolateA | Trichoderma harzianum | /path/to/proteins.faa | fusarium_graminearum |
        | isolateB | Trichoderma harzianum | /path/to/proteins.faa | fusarium_graminearum |

    - **iprscan**: Path to the `InterProScan` shell script.

- `resources`

    - **threads**: Maximum CPUs passed to individual tools inside rules.
    - **ram_gb**: Maximum RAM value used by memory-aware tools.

    `--cores` is Snakemake's scheduler limit. `resources: threads` controls tool-level thread arguments.

- `parameters`

    **Flye and Medaka**

    ```yaml
    flye_input_mode: auto
    medaka_model:
    ```

    For `_ont` samples, allowed `flye_input_mode` values are:

    - `auto`: default. Uses `--nano-hq`, unless an explicit Medaka model name contains `fast`, in which case it uses `--nano-raw`.
    - `nano-raw`: force Flye `--nano-raw`.
    - `nano-hq`: force Flye `--nano-hq`.
    - `pacbio-hifi`: force Flye `--pacbio-hifi` and skip Medaka for ONT-suffixed samples. This is retained as an explicit override. The recommended way to run PacBio HiFi samples is to use the `_pacbio` filename suffix, which lets the workflow select the correct PacBio-specific steps automatically.

    For `_pacbio` samples, `flye_input_mode` is ignored and Flye always receives `--pacbio-hifi`.

    Medaka behavior:

    - empty, null, `true`, or `auto`: run Medaka for eligible ONT samples and let Medaka infer a model from FASTQ headers;
    - explicit model string: validate the model against `medaka tools list_models` and use it;
    - `false`, `0`, `no`, or `off`: skip Medaka;
    - `_pacbio` suffix: skip Medaka regardless of `medaka_model`.

    **Decontamination**

    FunFluxL uses the same selector logic as FunFlux:

    ```yaml
    decontamination:
      mode: off
      discard_no_hit: true
      include_genera:
      include_genera_by_sample:
      exclude_genera:
      exclude_genera_file:
      sample_overrides:
    ```

    Available modes:

    - `off`: keep all contigs. `discard_no_hit` is ignored.
    - `auto`: keep the most abundant assigned genus.
    - `include`: keep only listed genera.
    - `exclude`: remove listed genera.

    `discard_no_hit: true` removes BLAST `no-hit` contigs only when the mode is `auto`, `include`, or `exclude`.
    In `auto` and `include` modes, the selector can treat selected genus aliases and retained legacy prefixes as equivalent. This is mainly a safeguard against false contig removal when BLAST/BlobTools assigns related or recently reclassified genera inconsistently. Although FunFluxL targets fungal genomes, bacterial genera may appear here because bacterial contamination can occur in fungal WGS assemblies. Alias-based decisions are recorded in `contig_taxonomy_decisions.tsv` with reasons such as `auto_genus_alias` or `included_genus_alias`. `exclude` mode remains exact. These aliases are heuristic safeguards, not a formal taxonomic reconciliation system.

    Genera can be supplied directly:

    ```yaml
    exclude_genera: Acidovorax;Pseudomonas;Sphingomonas
    ```

    or through a one-genus-per-line file:

    ```yaml
    exclude_genera_file: /path/to/exclude_genera.txt
    ```

    Optional sample overrides use a tab-separated file:

    ```text
    sample<TAB>mode<TAB>include_genera<TAB>exclude_genera<TAB>discard_no_hit
    isolateA<TAB>exclude<TAB><TAB>Acidovorax;Pseudomonas<TAB>true
    ```

    Use real tab characters, not the literal string `<TAB>`. A non-empty sample-specific include or exclude list replaces the corresponding global list for that sample; an empty include or exclude cell leaves the global list unchanged.

    If your BLAST database is stored below `directories.blast_db` under a name other than `core_nt`, set `nt_version`:

    ```yaml
    nt_version: core_nt
    ```

    **ITS taxonomy**

    ```yaml
    its_taxonomy_cutoff: 0.8
    ```

    **Repeat masking**

    ```yaml
    masking_method: tantan
    repeatmodeler_quick: true
    repeatmodeler_ltrstruct: false
    ```

    To use the advanced RepeatModeler + RepeatMasker strategy, change only `masking_method`:

    ```yaml
    masking_method: repeatmodeler_repeatmasker
    ```

    Available masking methods:

    - `tantan`: default lightweight softmasking.
    - `repeatmodeler_repeatmasker`: builds a sample-specific repeat library with `BuildDatabase` and `RepeatModeler`, then softmasks the contigs with `RepeatMasker -lib <RepeatModeler library> -xsmall`.

    `repeatmodeler_quick: true` adds `RepeatModeler -quick`. `repeatmodeler_ltrstruct: true` adds `RepeatModeler -LTRStruct`.

[⬆ Back to Table of Contents](#table-of-contents)

## Running FunFluxL
`FunFluxL` can be executed as a Snakemake workflow.

```bash
conda activate snakemake
snakemake --configfile config/config.yaml --sdm conda --cores 24 --jobs 1
```

If you resume an interrupted run:

```bash
snakemake --configfile config/config.yaml --unlock
snakemake --configfile config/config.yaml --sdm conda --cores 24 --jobs 1 --rerun-triggers mtime --rerun-incomplete
```

After a successful run, optional cleanup of bulky intermediate files can be inspected with:

```bash
workflow/scripts/clean_funflux_output.sh --target /path/to/output_dir
```

To actually remove the listed files:

```bash
workflow/scripts/clean_funflux_output.sh --run --target /path/to/output_dir
```

The cleanup script is dry-run by default and refuses targets that do not look like supported FunFlux, FunFluxL, or Funnotator output directories.

[⬆ Back to Table of Contents](#table-of-contents)

## Output
Here's a breakdown of the sub-directories created by `FunFluxL` within the main output folder.

```text
├── 01.pre-processing
├── 02.assembly
├── 03.post-processing
├── 04.annotation
├── logs
└── report
```

- `01.pre-processing`: Long reads filtered with [Filtlong](https://github.com/rrwick/Filtlong) v0.3.1. Raw and filtered read QC summaries are produced with [NanoPlot](https://github.com/wdecoster/NanoPlot) v1.46.2.

- `02.assembly`: Output from [Flye](https://github.com/mikolmogorov/Flye) v2.9.6. Key files include `assembly.fasta` and `assembly_linearized.fasta`.

- `03.post-processing`: Contains:
    - **mapping_evaluation**: [minimap2](https://github.com/lh3/minimap2) v2.30 and [samtools](https://github.com/samtools/samtools) v1.21 mapping output, with [QualiMap](http://qualimap.conesalab.org/) v2.3 mapping QC.
    - **contaminants**: BLAST+ v2.16.0 and BlobTools v1.1.1 decontamination output, including genus composition and `contig_taxonomy_decisions.tsv`.
    - **consensus**: [Medaka](https://github.com/nanoporetech/medaka) v2.2.2 consensus FASTA for ONT samples when Medaka is enabled. This directory is absent for PacBio HiFi samples and ONT samples with Medaka disabled.
    - **completeness_evaluation**: [BUSCO](https://busco.ezlab.org/) v6.0.0 output from `--auto-lineage-euk`.
    - **ITS_extraction**: [ITSx](https://microbiology.se/software/itsx/) v1.1.3 output and [VSEARCH](https://github.com/torognes/vsearch) v2.30.0 SINTAX classification against the automatically downloaded UNITE reference.

- `04.annotation`: Contains:
    - **repeatmasking**: Repeat masking output. With the default `tantan` mode, the softmasked contigs are written to `contigs_mask.fasta`. With `repeatmodeler_repeatmasker`, [RepeatModeler](https://github.com/Dfam-consortium/RepeatModeler) v2.0.8 and [RepeatMasker](https://www.repeatmasker.org/) v4.2.3 files are also kept inside each sample.
    - **iprscan**: [InterProScan](https://github.com/ebi-pf-team/interproscan) v5.77-108.0 XML output.
    - **eggnog**: [EggNOG-mapper](https://github.com/eggnogdb/eggnog-mapper) v2.1.13 annotation output.
    - **antismash**: [antiSMASH](https://github.com/antismash/antismash) v8.0.4 secondary metabolite output.
    - **funannotate**: Prediction and annotation directories from [funannotate](https://github.com/nextgenusfs/funannotate) v1.8.17.

- `report`: [MultiQC](https://github.com/MultiQC/MultiQC) v1.33 report aggregating NanoPlot, QualiMap, BUSCO, and other supported outputs.

[⬆ Back to Table of Contents](#table-of-contents)

## Acknowledgements
This work was supported by the [RATION project](https://www.ration-lrp.eu/project/) (Risk AssessmenT InnOvatioN for low-risk pesticides), funded by the European Union under Horizon Europe grant agreement No. [101084163](https://cordis.europa.eu/project/id/101084163).

## Citation
If you use `FunFluxL`, please cite:

Antonielli, L., & Brader, G. (2026). FunFluxL: Integrated workflow for fungal long-read genome assembly and annotation. Zenodo. https://doi.org/10.5281/zenodo.20796644

## References
01. Abarenkov, K., et al. (2024). The UNITE database for molecular identification and taxonomic communication of fungi and other eukaryotes: sequences, taxa and classifications reconsidered. Nucleic Acids Research, 52(D1), D791-D797. https://doi.org/10.1093/nar/gkad1039

02. Bengtsson-Palme, J., Ryberg, M., Hartmann, M., Branco, S., Wang, Z., Godhe, A., De Wit, P., Sánchez-García, M., Ebersberger, I., de Sousa, F., Amend, A., Jumpponen, A., Unterseher, M., Kristiansson, E., Abarenkov, K., Bertrand, Y. J. K., Sanli, K., Eriksson, K. M., Vik, U., ... Nilsson, R. H. (2013). Improved software detection and extraction of ITS1 and ITS2 from ribosomal ITS sequences of fungi and other eukaryotes for analysis of environmental sequencing data. Methods in Ecology and Evolution, 4(10), 914-919. https://doi.org/10.1111/2041-210X.12073

03. Blin, K., et al. (2025). antiSMASH 8.0: extended gene cluster detection capabilities and analyses of chemistry, enzymology and regulation. Nucleic Acids Research, 53(W1), W32-W38. https://doi.org/10.1093/nar/gkaf334

04. Borodovsky, M., & Lomsadze, A. (2011). Eukaryotic Gene Prediction Using GeneMark.hmm-E and GeneMark-ES. Current Protocols in Bioinformatics, Unit 4.6. https://doi.org/10.1002/0471250953.bi0406s35

05. Buchfink, B., Xie, C., & Huson, D. H. (2015). Fast and sensitive protein alignment using DIAMOND. Nature Methods, 12(1), 59-60. https://doi.org/10.1038/nmeth.3176

06. Camacho, C., Coulouris, G., Avagyan, V., Ma, N., Papadopoulos, J., Bealer, K., & Madden, T. L. (2009). BLAST+: Architecture and applications. BMC Bioinformatics, 10, 421. https://doi.org/10.1186/1471-2105-10-421

07. Cantalapiedra, C. P., Hernández-Plaza, A., Letunic, I., Bork, P., & Huerta-Cepas, J. (2021). eggNOG-mapper v2. Molecular Biology and Evolution, 38(12), 5825-5829. https://doi.org/10.1093/molbev/msab293

08. Challis, R., Richards, E., Rajan, J., Cochrane, G., & Blaxter, M. (2020). BlobToolKit - Interactive Quality Assessment of Genome Assemblies. G3 Genes|Genomes|Genetics, 10(4), 1361-1374. https://doi.org/10.1534/g3.119.400908

09. De Coster, W., D'Hert, S., Schultz, D. T., Cruts, M., & Van Broeckhoven, C. (2018). NanoPack: visualizing and processing long-read sequencing data. Bioinformatics, 34(15), 2666-2669. https://doi.org/10.1093/bioinformatics/bty149

10. Edgar, R. C. (2016). SINTAX: A simple non-Bayesian taxonomy classifier for 16S and ITS sequences. bioRxiv. https://doi.org/10.1101/074161

11. Ewels, P., Magnusson, M., Lundin, S., & Käller, M. (2016). MultiQC. Bioinformatics, 32(19), 3047-3048. https://doi.org/10.1093/bioinformatics/btw354

12. Flynn, J. M., Hubley, R., Goubert, C., Rosen, J., Clark, A. G., Feschotte, C., & Smit, A. F. (2020). RepeatModeler2 for automated genomic discovery of transposable element families. Proceedings of the National Academy of Sciences, 117(17), 9451-9457. https://doi.org/10.1073/pnas.1921046117

13. Frith, M. C. (2011). A new repeat-masking method enables specific detection of homologous sequences. Nucleic Acids Research, 39(4), e23. https://doi.org/10.1093/nar/gkq1212

14. Haas, B. J., Salzberg, S. L., Zhu, W., Pertea, M., Allen, J. E., Orvis, J., White, O., Buell, C. R., & Wortman, J. R. (2008). Automated eukaryotic gene structure annotation using EVidenceModeler and the Program to Assemble Spliced Alignments. Genome Biology, 9(1), R7. https://doi.org/10.1186/gb-2008-9-1-r7

15. Huerta-Cepas, J., Szklarczyk, D., Heller, D., Hernández-Plaza, A., Forslund, S. K., Cook, H., Mende, D. R., Letunic, I., Rattei, T., Jensen, L. J., von Mering, C., & Bork, P. (2019). eggNOG 5.0. Nucleic Acids Research, 47(D1), D309-D314. https://doi.org/10.1093/nar/gky1085

16. Jonathan M. Palmer, & Jason Stajich. (2020). Funannotate v1.8.1: Eukaryotic genome annotation [Computer software]. Zenodo. https://doi.org/10.5281/zenodo.4054262

17. Jones, P., Binns, D., Chang, H.-Y., Fraser, M., Li, W., McAnulla, C., McWilliam, H., Maslen, J., Mitchell, A., Nuka, G., Pesseat, S., Quinn, A. F., Sangrador-Vegas, A., Scheremetjew, M., Yong, S.-Y., Lopez, R., & Hunter, S. (2014). InterProScan 5. Bioinformatics, 30(9), 1236-1240. https://doi.org/10.1093/bioinformatics/btu031

18. Kolmogorov, M., Yuan, J., Lin, Y., & Pevzner, P. A. (2019). Assembly of long, error-prone reads using repeat graphs. Nature Biotechnology, 37, 540-546. https://doi.org/10.1038/s41587-019-0072-8

19. Köster, J., & Rahmann, S. (2012). Snakemake - A scalable bioinformatics workflow engine. Bioinformatics, 28(19), 2520-2522. https://doi.org/10.1093/bioinformatics/bts480

20. Li, H. (2018). Minimap2: pairwise alignment for nucleotide sequences. Bioinformatics, 34(18), 3094-3100. https://doi.org/10.1093/bioinformatics/bty191

21. Li, H., Handsaker, B., Wysoker, A., Fennell, T., Ruan, J., Homer, N., Marth, G., Abecasis, G., Durbin, R., & 1000 Genome Project Data Processing Subgroup. (2009). The Sequence Alignment/Map format and SAMtools. Bioinformatics, 25(16), 2078-2079. https://doi.org/10.1093/bioinformatics/btp352

22. Medaka. Oxford Nanopore Technologies. https://github.com/nanoporetech/medaka

23. Okonechnikov, K., Conesa, A., & García-Alcalde, F. (2016). Qualimap 2. Bioinformatics, 32(2), 292-294. https://doi.org/10.1093/bioinformatics/btv566

24. Rawlings, N. D., Waller, M., Barrett, A. J., & Bateman, A. (2014). MEROPS. Nucleic Acids Research, 42(D1), D503-D509. https://doi.org/10.1093/nar/gkt953

25. Rognes, T., Flouri, T., Nichols, B., Quince, C., & Mahé, F. (2016). VSEARCH. PeerJ, 4, e2584. https://doi.org/10.7717/peerj.2584

26. Smit, A. F. A., Hubley, R., & Green, P. RepeatMasker Open-4.0. http://www.repeatmasker.org

27. Stanke, M., Keller, O., Gunduz, I., Hayes, A., Waack, S., & Morgenstern, B. (2006). AUGUSTUS. Nucleic Acids Research, 34(Web Server issue), W435-W439. https://doi.org/10.1093/nar/gkl200

28. Tegenfeldt, F., Kuznetsov, D., Manni, M., Berkeley, M., Zdobnov, E. M., & Kriventseva, E. V. (2025). OrthoDB and BUSCO update: annotation of orthologs with wider sampling of genomes. Nucleic Acids Research, 53(D1), D516-D522. https://doi.org/10.1093/nar/gkae987

29. The UniProt Consortium. (2023). UniProt: The Universal Protein Knowledgebase in 2023. Nucleic Acids Research, 51(D1), D523-D531. https://doi.org/10.1093/nar/gkac1052

30. Wick, R. Filtlong. https://github.com/rrwick/Filtlong

31. Zheng, J., Ge, Q., Yan, Y., Zhang, X., Huang, L., & Yin, Y. (2023). dbCAN3: Automated carbohydrate-active enzyme and substrate annotation. Nucleic Acids Research, 51(W1), W115-W121. https://doi.org/10.1093/nar/gkad328
