# 🧬 AI + Genomics for EDS Research: 12-Week Study Plan

> **Learner Profile:** AWS Solutions Architect (all 14 AWS certifications, Anthropic certifications) with biology/chemistry background
> **Goal:** Build expertise in computational genomics and AI-driven variant interpretation, specifically for Ehlers-Danlos Syndrome research
> **Created:** September 2026

---

## 📋 Plan Overview

| Phase | Weeks | Theme | Key Deliverable |
|-------|-------|-------|-----------------|
| **Foundation** | 1–2 | Bioinformatics Fundamentals | Complete Rosalind problem sets; run Galaxy workflows |
| **Pipelines** | 3–4 | Genomics Pipelines | End-to-end variant calling from FASTQ → annotated VCF |
| **Cloud** | 5–6 | AWS HealthOmics & Cloud Genomics | Deploy Ready2Run workflows; build HealthOmics pipelines |
| **AI Interpretation** | 7–8 | AI Variant Interpretation | Score EDS variants with AlphaMissense, CADD, SpliceAI, REVEL |
| **Build** | 9–10 | EDS Variant Classifier | Train and evaluate an ML model on ClinVar EDS data via SageMaker |
| **Advanced** | 11–12 | Multi-Omics, Graph ML & Agents | Neptune knowledge graph; Bedrock AgentCore variant agent |

**Weekly commitment:** ~12–15 hours (evenings + weekends)

---

## Phase 1: Bioinformatics Foundations

### Week 1 — Biological Data & File Formats

**🎯 Learning Goal:** Understand core bioinformatics data types, file formats (FASTA, FASTQ, SAM/BAM, VCF, BED, GFF), and the biological context of DNA sequencing.

#### Resources
| Resource | URL | Type | Time |
|----------|-----|------|------|
| Rosalind — Bioinformatics Stronghold | [rosalind.info/problems/list-view](https://rosalind.info/problems/list-view/) | Interactive problems | 4h |
| Galaxy Training — Introduction to Genomics | [training.galaxyproject.org](https://training.galaxyproject.org/) | Hands-on tutorials | 3h |
| SAM/BAM Format Specification | [samtools.github.io/hts-specs](https://samtools.github.io/hts-specs/) | Reference docs | 1h |
| VCF Format Specification (v4.3+) | [samtools.github.io/hts-specs/VCFv4.3.pdf](https://samtools.github.io/hts-specs/VCFv4.3.pdf) | Reference docs | 1h |
| UCSC Genome Browser Tutorial | [genome.ucsc.edu](https://genome.ucsc.edu/) | Web tool | 1h |

#### Rosalind Problems to Complete (Week 1)
1. **DNA** — Counting DNA Nucleotides
2. **RNA** — Transcribing DNA into RNA
3. **REVC** — Complementing a Strand of DNA
4. **GC** — Computing GC Content
5. **HAMM** — Counting Point Mutations
6. **PROT** — Translating RNA into Protein
7. **SUBS** — Finding a Motif in DNA
8. **CONS** — Consensus and Profile

#### Hands-On Exercise (⏱ 4 hours)
- **Set up your bioinformatics environment:**
  - Install `samtools`, `bcftools`, `bedtools` via conda/mamba
  - Download a sample BAM file from the 1000 Genomes Project on AWS: `s3://1000genomes/`
  - Practice: convert BAM → SAM, extract reads for a specific region (e.g., COL5A1 on chr9), view in IGV
  - Parse a VCF file with `bcftools query` to extract variant positions and allele frequencies

#### Key Concepts to Master
- [ ] Central dogma: DNA → RNA → Protein
- [ ] What PHRED quality scores mean (Q30 = 1 in 1000 error rate)
- [ ] Difference between reference genome builds (GRCh37/hg19 vs GRCh38/hg38)
- [ ] FASTA vs FASTQ: when each is used
- [ ] SAM flags and CIGAR strings
- [ ] VCF structure: CHROM, POS, REF, ALT, QUAL, FILTER, INFO, FORMAT

---

### Week 2 — Galaxy Project & Sequencing Technologies

**🎯 Learning Goal:** Run complete analysis workflows on Galaxy; understand NGS technologies (Illumina short-read, PacBio/ONT long-read) and their tradeoffs for EDS research.

#### Resources
| Resource | URL | Type | Time |
|----------|-----|------|------|
| Galaxy Training — Variant Analysis | [training.galaxyproject.org/topics/variant-analysis](https://training.galaxyproject.org/training-material/topics/variant-analysis/) | Hands-on tutorial | 4h |
| Galaxy Training — Sequence Analysis | [training.galaxyproject.org/topics/sequence-analysis](https://training.galaxyproject.org/training-material/topics/sequence-analysis/) | Hands-on tutorial | 3h |
| Illumina Sequencing Technology Overview | [illumina.com/science/technology](https://www.illumina.com/science/technology/next-generation-sequencing.html) | Documentation | 1h |
| StatQuest — NGS Explained (YouTube) | [youtube.com/@statquest](https://www.youtube.com/@statquest) | Video | 1h |
| Heng Li's Blog — Sequencing Costs & Quality | [lh3.github.io](https://lh3.github.io/) | Blog | 1h |

#### Rosalind Problems to Complete (Week 2)
9. **LCSM** — Finding a Shared Motif
10. **MPRT** — Finding a Protein Motif
11. **SPLC** — RNA Splicing
12. **REVP** — Locating Restriction Sites
13. **LONG** — Genome Assembly as Shortest Superstring

#### Hands-On Exercise (⏱ 5 hours)
- **Run the Galaxy Variant Analysis tutorial end-to-end:**
  - Upload sample FASTQ reads to [usegalaxy.org](https://usegalaxy.org/)
  - Run FastQC for quality assessment
  - Map reads with BWA-MEM
  - Call variants with FreeBayes
  - Annotate with SnpEff
  - Visualize in the Galaxy built-in genome browser
- **EDS Context:** Search the UCSC Genome Browser for EDS-related genes:
  - *COL5A1* (chr9), *COL5A2* (chr2), *COL3A1* (chr2), *COL1A1* (chr17), *COL1A2* (chr7)
  - *TNXB* (chr6), *ADAMTS2* (chr5), *PLOD1* (chr1), *FKBP14* (chr7), *COL12A1* (chr6)

#### Key Concepts to Master
- [ ] Short-read vs long-read sequencing: coverage, accuracy, cost tradeoffs
- [ ] WGS vs WES vs targeted panel sequencing — which is best for EDS
- [ ] Read mapping: seed-and-extend algorithms
- [ ] Galaxy workflow system: how tools chain together
- [ ] Quality control metrics: duplication rate, mapping quality, on-target rate

---

## Phase 2: Genomics Pipelines

### Week 3 — Read Alignment & Variant Calling

**🎯 Learning Goal:** Build a local variant calling pipeline using BWA-MEM2, samtools, and GATK. Understand the GATK Best Practices workflow for germline short variant discovery.

#### Resources
| Resource | URL | Type | Time |
|----------|-----|------|------|
| BWA-MEM2 GitHub | [github.com/bwa-mem2/bwa-mem2](https://github.com/bwa-mem2/bwa-mem2) | Tool + docs | 2h |
| GATK Best Practices — Germline SNPs + Indels | [gatk.broadinstitute.org/hc/en-us/articles/360035535932](https://gatk.broadinstitute.org/hc/en-us/articles/360035535932-Germline-short-variant-discovery-SNPs-Indels) | Documentation | 3h |
| GATK Tutorials (Broad Institute) | [gatk.broadinstitute.org/hc/en-us/sections/360007226651](https://gatk.broadinstitute.org/hc/en-us/sections/360007226651-Tutorials) | Hands-on | 3h |
| DeepVariant — Google GitHub | [github.com/google/deepvariant](https://github.com/google/deepvariant) | Tool + docs | 2h |
| DeepVariant Case Study | [google.github.io/deepvariant](https://google.github.io/deepvariant/) | Tutorial | 2h |

#### Hands-On Exercise (⏱ 6 hours)
- **Build the pipeline locally (Docker recommended):**
  ```bash
  # 1. Index reference genome
  bwa-mem2 index GRCh38.fa

  # 2. Align reads
  bwa-mem2 mem -t 8 -R "@RG\tID:sample1\tSM:sample1\tPL:ILLUMINA" \
    GRCh38.fa sample_R1.fq.gz sample_R2.fq.gz | \
    samtools sort -o aligned.bam

  # 3. Mark duplicates
  gatk MarkDuplicates -I aligned.bam -O deduped.bam -M metrics.txt

  # 4. Base quality score recalibration (BQSR)
  gatk BaseRecalibrator -R GRCh38.fa -I deduped.bam \
    --known-sites dbsnp_146.hg38.vcf.gz -O recal_data.table
  gatk ApplyBQSR -R GRCh38.fa -I deduped.bam \
    --bqsr-recal-file recal_data.table -O recalibrated.bam

  # 5. Call variants with HaplotypeCaller
  gatk HaplotypeCaller -R GRCh38.fa -I recalibrated.bam \
    -O output.g.vcf.gz -ERC GVCF
  ```
- **Also run DeepVariant on the same sample for comparison:**
  ```bash
  docker run google/deepvariant:latest \
    /opt/deepvariant/bin/run_deepvariant \
    --model_type=WGS --ref=GRCh38.fa \
    --reads=recalibrated.bam --output_vcf=dv_output.vcf.gz
  ```
- **Compare GATK vs DeepVariant calls** using `bcftools isec`
- **Use GIAB truth set** (HG002) for benchmarking with `hap.py`

#### Key Concepts to Master
- [ ] BWA-MEM2 algorithm: FM-index, seed extension, SIMD acceleration
- [ ] Why marking duplicates matters (PCR artifacts)
- [ ] BQSR: correcting systematic errors in base quality scores
- [ ] HaplotypeCaller: local de novo assembly of haplotypes
- [ ] DeepVariant: CNN approach to variant calling (pileup images → genotype)
- [ ] GVCF mode: why it enables scalable joint calling

---

### Week 4 — Variant Annotation & EDS Gene Focus

**🎯 Learning Goal:** Annotate variants using VEP, filter for EDS-relevant genes, understand variant classification standards (ACMG/AMP), and explore ClinVar for EDS.

#### Resources
| Resource | URL | Type | Time |
|----------|-----|------|------|
| Ensembl VEP Documentation | [ensembl.org/info/docs/tools/vep](https://www.ensembl.org/info/docs/tools/vep/index.html) | Documentation | 2h |
| Ensembl VEP Web Interface | [ensembl.org/Tools/VEP](https://www.ensembl.org/Tools/VEP) | Web tool | 1h |
| VEP GitHub — Command-line install | [github.com/Ensembl/ensembl-vep](https://github.com/Ensembl/ensembl-vep) | Tool | 1h |
| ACMG/AMP Variant Classification Standards | [nature.com/articles/gim201530](https://www.nature.com/articles/gim201530) | Paper | 2h |
| ClinVar — EDS Variants (COL5A1) | [ncbi.nlm.nih.gov/clinvar/?term=COL5A1+ehlers+danlos](https://www.ncbi.nlm.nih.gov/clinvar/?term=COL5A1%5Bgene%5D+AND+ehlers+danlos) | Database | 2h |
| SnpEff & SnpSift | [pcingola.github.io/SnpEff](https://pcingola.github.io/SnpEff/) | Tool + docs | 1h |

#### Hands-On Exercise (⏱ 6 hours)
- **Annotate your VCF with VEP:**
  ```bash
  vep -i output.vcf.gz -o annotated.vcf \
    --cache --assembly GRCh38 --vcf \
    --sift b --polyphen b --af --af_gnomade \
    --plugin CADD,whole_genome_SNVs.tsv.gz \
    --plugin SpliceAI,snv=spliceai_scores.raw.snv.hg38.vcf.gz
  ```
- **Filter for EDS genes:**
  ```bash
  # Define EDS gene panel
  EDS_GENES="COL5A1,COL5A2,COL3A1,COL1A1,COL1A2,TNXB,COL12A1,ADAMTS2,PLOD1,FKBP14,B4GALT7,B3GALT6,SLC39A13,AEBP1,CHST14,DSE"

  bcftools view -i "INFO/CSQ ~ 'COL5A1' || INFO/CSQ ~ 'COL5A2' || INFO/CSQ ~ 'COL3A1'" \
    annotated.vcf > eds_variants.vcf
  ```
- **Download and explore ClinVar EDS data:**
  - Visit [ClinVar FTP](https://ftp.ncbi.nlm.nih.gov/pub/clinvar/) and download `clinvar.vcf.gz`
  - Filter for EDS conditions and EDS gene panel
  - Tabulate: how many Pathogenic / Likely Pathogenic / VUS / Benign per gene

#### Key Concepts to Master
- [ ] VEP consequence types: missense, nonsense, frameshift, splice donor/acceptor
- [ ] ACMG/AMP classification: Pathogenic, Likely Pathogenic, VUS, Likely Benign, Benign
- [ ] Evidence codes: PVS1, PS1-PS4, PM1-PM6, PP1-PP5 (and benign equivalents)
- [ ] EDS genetics: autosomal dominant (classic, vascular) vs recessive subtypes
- [ ] Collagen structure: triple helix, Gly-X-Y repeat, why glycine substitutions are pathogenic
- [ ] Haploinsufficiency vs dominant-negative mechanisms in collagenopathies

---

## Phase 3: AWS HealthOmics & Cloud Genomics

### Week 5 — AWS HealthOmics Setup & Ready2Run Workflows

**🎯 Learning Goal:** Deploy AWS HealthOmics for genomics workloads. Run Ready2Run workflows (GATK, nf-core). Understand HealthOmics storage, sequence stores, and workflow engines.

#### Resources
| Resource | URL | Type | Time |
|----------|-----|------|------|
| AWS HealthOmics Documentation | [docs.aws.amazon.com/omics](https://docs.aws.amazon.com/omics/latest/dev/what-is-service.html) | Documentation | 3h |
| HealthOmics Getting Started | [docs.aws.amazon.com/omics/latest/dev/getting-started](https://docs.aws.amazon.com/omics/latest/dev/getting-started.html) | Tutorial | 2h |
| Ready2Run Workflows Catalog | [docs.aws.amazon.com/omics/latest/dev/workflows-r2r-table](https://docs.aws.amazon.com/omics/latest/dev/workflows-r2r-table.html) | Reference | 1h |
| AWS HealthOmics Tutorials (GitHub) | [github.com/aws-samples/amazon-omics-tutorials](https://github.com/aws-samples/amazon-omics-tutorials/) | Jupyter notebooks | 4h |
| Amazon Omics End-to-End Genomics | [github.com/aws-samples/amazon-omics-end-to-end-genomics](https://github.com/aws-samples/amazon-omics-end-to-end-genomics) | Sample code | 2h |

#### Hands-On Exercise (⏱ 7 hours)
- **Set up HealthOmics infrastructure:**
  ```bash
  # Create a reference store
  aws omics create-reference-store --name eds-reference-store

  # Import GRCh38 reference
  aws omics start-reference-import-job \
    --reference-store-id <store-id> \
    --sources '[{"sourceFile":"s3://broad-references/hg38/v0/Homo_sapiens_assembly38.fasta"}]'

  # Create a sequence store
  aws omics create-sequence-store --name eds-sequence-store

  # Create a variant store for annotated VCFs
  aws omics create-variant-store --name eds-variants \
    --reference '{"referenceArn":"<ref-arn>"}'
  ```
- **Run a Ready2Run GATK Best Practices workflow:**
  ```bash
  aws omics start-run \
    --workflow-id <gatk-r2r-workflow-id> \
    --workflow-type READY2RUN \
    --role-arn <omics-role-arn> \
    --parameters '{"sample_name":"HG002","input_cram":"s3://..."}' \
    --output-uri s3://my-omics-output/
  ```
- **Monitor the run:** Use the HealthOmics console Run Dashboard
- **Import results into the Variant Store** and query with `aws omics get-variant-store`

#### Key Concepts to Master
- [ ] HealthOmics three pillars: Storage (sequence/reference/variant/annotation stores), Workflows, Analytics
- [ ] Ready2Run vs Private workflows: when to use each
- [ ] WDL vs Nextflow vs CWL: workflow languages supported
- [ ] HealthOmics pricing model: run storage, compute, data storage
- [ ] Integration with S3, ECR (Docker containers), and IAM roles
- [ ] Annotation stores: importing ClinVar, gnomAD into HealthOmics

---

### Week 6 — Cloud-Scale Genomics & AWS Genomics Ecosystem

**🎯 Learning Goal:** Build custom HealthOmics workflows, integrate with the broader AWS ecosystem (S3 Tables, Athena, Lake Formation), and explore aws-samples genomics repositories.

#### Resources
| Resource | URL | Type | Time |
|----------|-----|------|------|
| HealthOmics + Bedrock AgentCore Blog | [aws.amazon.com/blogs/machine-learning/accelerating-genomics-variant-interpretation-with-aws-healthomics-and-amazon-bedrock-agentcore](https://aws.amazon.com/blogs/machine-learning/accelerating-genomics-variant-interpretation-with-aws-healthomics-and-amazon-bedrock-agentcore/) | Blog post | 2h |
| AWS Automated Genomics Processing | [github.com/aws-samples/aws-healthomics-automated-genomics-processing](https://github.com/aws-samples/aws-healthomics-automated-genomics-processing) | Sample code | 2h |
| Multi-Omics Data Integration on AWS | [github.com/awslabs/guidance-for-multi-omics-and-multi-modal-data-integration-and-analysis-on-aws](https://github.com/awslabs/genomics-tertiary-analysis-and-data-lakes-using-aws-glue-and-amazon-athena) | Architecture guide | 2h |
| Amazon Omics Analysis App | [github.com/aws-samples/amazon-omics-analysis-app](https://github.com/aws-samples/amazon-omics-analysis-app) | Full-stack sample | 2h |
| 1000 Genomes on AWS Open Data | [registry.opendata.aws/1000-genomes](https://registry.opendata.aws/1000-genomes/) | Dataset | 1h |
| 1000 Genomes Reanalysis (2026) | [aws.amazon.com/blogs/publicsector/the-1000-genomes-project-reanalyzed](https://aws.amazon.com/blogs/publicsector/the-1000-genomes-project-reanalyzed-a-new-analytical-baseline-for-human-genomics/) | Blog post | 1h |

#### Hands-On Exercise (⏱ 7 hours)
- **Write a custom WDL workflow** for EDS-targeted analysis:
  ```wdl
  version 1.0

  workflow EDS_VariantAnalysis {
    input {
      File input_bam
      File reference_fasta
      File eds_gene_bed  # BED file of EDS gene regions
    }

    call ExtractEDSRegions { input: bam=input_bam, bed=eds_gene_bed }
    call CallVariants { input: bam=ExtractEDSRegions.output_bam, ref=reference_fasta }
    call AnnotateVariants { input: vcf=CallVariants.output_vcf }
  }
  ```
- **Deploy the workflow to HealthOmics** and run on 1000 Genomes sample data
- **Set up an S3 Tables iceberg table** for storing variant annotations queryable by Athena:
  ```sql
  -- Query EDS variants across samples
  SELECT sample_id, gene, hgvs_c, hgvs_p, clinvar_clnsig,
         gnomad_af, cadd_phred
  FROM eds_variant_annotations
  WHERE gene IN ('COL5A1', 'COL5A2', 'COL3A1', 'TNXB')
    AND cadd_phred > 20
  ORDER BY cadd_phred DESC;
  ```
- **Explore the AWS Genomics CLI deprecation notice** and understand migration to HealthOmics

#### Key Concepts to Master
- [ ] Writing WDL workflows: tasks, calls, scatter-gather patterns
- [ ] S3 Tables (Iceberg format) for genomic data lakes
- [ ] Athena + Lake Formation for serverless genomics queries
- [ ] Cost optimization: spot instances, run storage sizing, tiered storage
- [ ] Genomics data governance: HIPAA considerations, data encryption

---

## Phase 4: AI Variant Interpretation

### Week 7 — Pathogenicity Prediction Tools

**🎯 Learning Goal:** Understand and apply state-of-the-art AI tools for variant pathogenicity prediction: AlphaMissense, CADD, SpliceAI, and REVEL. Score EDS-specific variants.

#### Resources
| Resource | URL | Type | Time |
|----------|-----|------|------|
| AlphaMissense — Google DeepMind | [deepmind.google/research/publications/21083](https://deepmind.google/research/publications/21083/) | Paper + data | 3h |
| AlphaMissense Predictions Download | [zenodo.org/records/8208688](https://zenodo.org/records/8208688) | Dataset (71M variants) | 1h |
| CADD v1.7 (GRCh38) | [cadd.bihealth.org](https://cadd.bihealth.org/) | Web tool + API | 2h |
| CADD v1.7 Paper | [academic.oup.com/nar/article/52/D1/D1143/7511313](https://academic.oup.com/nar/article/52/D1/D1143/7511313) | Paper | 1h |
| SpliceAI — Illumina | [github.com/illumina/spliceAI](https://github.com/illumina/spliceAI) | Tool + code | 2h |
| SpliceAI Paper (Cell 2019) | [cell.com/cell/fulltext/S0092-8674(18)31629-5](https://www.cell.com/cell/fulltext/S0092-8674(18)31629-5) | Paper | 1h |
| REVEL — Ensemble Pathogenicity Predictor | [sites.google.com/site/revelgenomics](https://sites.google.com/site/revelgenomics/) | Tool + scores | 1h |
| AlphaGenome — DeepMind (2026) | [nature.com/articles/s41586-025-10014-0](https://www.nature.com/articles/s41586-025-10014-0) | Paper | 1h |

#### Hands-On Exercise (⏱ 7 hours)
- **Score EDS variants with all four tools:**
  1. **Download ClinVar EDS variants** (Pathogenic + VUS for COL5A1, COL5A2, COL3A1, TNXB):
     ```python
     import pandas as pd

     # Load ClinVar data
     clinvar = pd.read_csv('clinvar_variant_summary.txt.gz', sep='\t')
     eds_genes = ['COL5A1', 'COL5A2', 'COL3A1', 'COL1A1', 'COL1A2',
                  'TNXB', 'COL12A1', 'ADAMTS2', 'PLOD1', 'FKBP14']
     eds_variants = clinvar[clinvar['GeneSymbol'].isin(eds_genes)]
     ```
  2. **Run CADD** on the variant list via the web API or pre-scored files
  3. **Run SpliceAI** locally:
     ```bash
     spliceai -I eds_variants.vcf -O eds_spliceai.vcf \
       -R GRCh38.fa -A grch38
     ```
  4. **Look up AlphaMissense scores** from the pre-computed database
  5. **Look up REVEL scores** from the tabixed score file
- **Create a comparison table:** For each EDS variant, compile scores from all tools
- **Identify discordances:** Which VUS variants have high pathogenicity scores? These are high-priority for reclassification

#### Key Concepts to Master
- [ ] AlphaMissense: AlphaFold-based, trained on population frequency, structural context
- [ ] CADD: meta-predictor integrating 60+ annotations, PHRED-scaled (>20 = top 1%)
- [ ] SpliceAI: 32-layer ResNet predicting splice site creation/disruption, delta scores
- [ ] REVEL: ensemble of 13 tools (MutPred, VEST3, PolyPhen-2, SIFT, etc.)
- [ ] Why no single tool is sufficient: each captures different variant effects
- [ ] Calibration vs discrimination: what these scores mean clinically
- [ ] Collagen-specific considerations: Gly-X-Y violations, triple helix disruption

---

### Week 8 — Advanced Interpretation & EDS-Specific Analysis

**🎯 Learning Goal:** Integrate multiple evidence sources for clinical-grade variant interpretation. Understand the EDS genetic landscape and current diagnostic challenges (especially hEDS).

#### Resources
| Resource | URL | Type | Time |
|----------|-----|------|------|
| Shirvani et al. — hEDS ML Study (MDPI Genes 2026) | [mdpi.com/2073-4425/17/2/211](https://www.mdpi.com/2073-4425/17/2/211) | Paper | 2h |
| aiDIVA — Hybrid AI for Rare Disease (Nature 2026) | [nature.com/articles/s41525-026-00611-x](https://www.nature.com/articles/s41525-026-00611-x) | Paper | 2h |
| GenPhenia — DNN for Rare Disease Diagnosis (2026) | [springer.com/article/10.1186/s40246-026-01023-9](https://link.springer.com/article/10.1186/s40246-026-01023-9) | Paper | 2h |
| G.AI Platform (J Translational Medicine 2026) | [springer.com/article/10.1186/s12967-026-08368-8](https://link.springer.com/article/10.1186/s12967-026-08368-8) | Paper | 2h |
| InterVar — ACMG Automated Interpretation | [wintervar.wglab.org](https://wintervar.wglab.org/) | Web tool | 1h |
| ClinGen EDS Expert Panel | [clinicalgenome.org](https://www.clinicalgenome.org/) | Database | 1h |

#### Hands-On Exercise (⏱ 6 hours)
- **Replicate key findings from Shirvani et al.:**
  - Download their supplementary data
  - Understand their ML approach: which features distinguished hEDS subjects
  - Analyze the gene networks identified (extracellular matrix, cell adhesion, immune)
- **Apply aiDIVA to EDS variants:**
  - Install aiDIVA from GitHub
  - Run on a test VCF with EDS-relevant variants
  - Compare aiDIVA rankings vs CADD/REVEL/AlphaMissense
- **Build an evidence integration spreadsheet:**
  - For 50 EDS VUS variants, compile:
    - Population frequency (gnomAD)
    - All pathogenicity scores (CADD, REVEL, SpliceAI, AlphaMissense)
    - Functional domain (Gly-X-Y, propeptide, crosslinking site)
    - Protein structure impact (AlphaFold-predicted)
    - In silico ACMG code assignment
  - Flag variants likely to be reclassified as LP/P

#### Key Concepts to Master
- [ ] hEDS vs classical EDS: why hEDS lacks a known single gene (polygenic hypothesis)
- [ ] The Shirvani et al. approach: multi-system ML capturing oligogenic architecture
- [ ] aiDIVA: combining random forest, evidence-based scoring, and LLM reasoning
- [ ] GenPhenia: deep neural network for phenotype-driven gene prioritization
- [ ] Phenotype-driven analysis: using HPO terms to prioritize candidate variants
- [ ] The diagnostic odyssey: average 5–7 years to EDS diagnosis
- [ ] Limitations of current tools: bias toward well-studied variants and European populations

---

## Phase 5: Build the EDS Variant Classifier

### Week 9 — Data Engineering & Feature Design

**🎯 Learning Goal:** Download and process ClinVar data for EDS genes. Engineer features combining sequence properties, structural predictions, population frequencies, and conservation scores. Prepare training data for ML.

#### Resources
| Resource | URL | Type | Time |
|----------|-----|------|------|
| ClinVar FTP Download | [ftp.ncbi.nlm.nih.gov/pub/clinvar/](https://ftp.ncbi.nlm.nih.gov/pub/clinvar/) | Dataset | 1h |
| ClinVar Submission Summary | [ncbi.nlm.nih.gov/clinvar/](https://www.ncbi.nlm.nih.gov/clinvar/) | Database | 1h |
| SageMaker Studio Setup | [docs.aws.amazon.com/sagemaker](https://docs.aws.amazon.com/sagemaker/latest/dg/gs-studio.html) | Documentation | 1h |
| SageMaker Processing Jobs | [docs.aws.amazon.com/sagemaker/latest/dg/processing-job](https://docs.aws.amazon.com/sagemaker/latest/dg/processing-job.html) | Documentation | 1h |
| AstraZeneca Genomics FM on SageMaker | [aws.amazon.com/blogs/industries/astrazeneca-fine-tunes-genomics-foundation-models-with-amazon-sagemaker](https://aws.amazon.com/blogs/industries/astrazeneca-fine-tunes-genomics-foundation-models-with-amazon-sagemaker/) | Blog | 1h |
| Genomics England on SageMaker | [aws.amazon.com/blogs/machine-learning/genomics-england-uses-amazon-sagemaker](https://aws.amazon.com/blogs/machine-learning/genomics-england-uses-amazon-sagemaker-to-predict-cancer-subtypes-and-patient-survival-from-multi-modal-data/) | Blog | 1h |

#### Hands-On Exercise (⏱ 8 hours)
- **Data acquisition and cleaning:**
  ```python
  import pandas as pd
  import boto3

  # Download ClinVar
  # ftp.ncbi.nlm.nih.gov/pub/clinvar/tab_delimited/variant_summary.txt.gz
  clinvar = pd.read_csv('variant_summary.txt.gz', sep='\t')

  # Filter to connective tissue / EDS genes
  eds_genes = ['COL5A1','COL5A2','COL3A1','COL1A1','COL1A2','TNXB',
               'COL12A1','ADAMTS2','PLOD1','FKBP14','B4GALT7','B3GALT6',
               'SLC39A13','AEBP1','CHST14','DSE','COL6A1','COL6A2','COL6A3',
               'FLNA','ZNF469','PRDM5']
  eds_df = clinvar[clinvar['GeneSymbol'].isin(eds_genes)].copy()

  # Binarize: Pathogenic/Likely_pathogenic=1, Benign/Likely_benign=0
  # Remove VUS for training (or use as holdout for prediction)
  label_map = {
      'Pathogenic': 1, 'Likely pathogenic': 1,
      'Pathogenic/Likely pathogenic': 1,
      'Benign': 0, 'Likely benign': 0,
      'Benign/Likely benign': 0
  }
  eds_labeled = eds_df[eds_df['ClinicalSignificance'].isin(label_map.keys())]
  eds_labeled['label'] = eds_labeled['ClinicalSignificance'].map(label_map)
  ```
- **Feature engineering pipeline (run as SageMaker Processing Job):**
  - **Sequence features:** variant type, amino acid change, Gly-X-Y position, codon change
  - **Conservation:** phyloP, phastCons, GERP++
  - **Population frequency:** gnomAD AF (overall, per population)
  - **Pathogenicity scores:** CADD, REVEL, SpliceAI delta scores, AlphaMissense
  - **Structural features:** protein domain (InterPro), secondary structure, solvent accessibility
  - **Constraint metrics:** pLI, LOEUF, Z-scores from gnomAD per gene
- **Split data:** 70% train, 15% validation, 15% test (stratified by gene and label)
- **Upload to S3** in SageMaker-compatible format

#### Key Concepts to Master
- [ ] Class imbalance in variant data: more benign than pathogenic for some genes
- [ ] Data leakage risks: don't use ClinVar submitter information as features
- [ ] Feature importance: which features are most predictive for collagen variants
- [ ] Circular evidence: avoid using tool scores trained on the same ClinVar data
- [ ] Train/test contamination: temporal splitting (train on older submissions, test on newer)
- [ ] Handling missing data: imputation strategies for sparse annotation fields

---

### Week 10 — Model Training, Evaluation & Deployment

**🎯 Learning Goal:** Train, evaluate, and deploy an EDS variant classifier on SageMaker. Compare XGBoost, neural network, and ensemble approaches. Evaluate on held-out VUS variants.

#### Resources
| Resource | URL | Type | Time |
|----------|-----|------|------|
| SageMaker XGBoost Algorithm | [docs.aws.amazon.com/sagemaker/latest/dg/xgboost](https://docs.aws.amazon.com/sagemaker/latest/dg/xgboost.html) | Documentation | 1h |
| SageMaker Training Tutorial | [aws.amazon.com/tutorials/build-train-deploy-monitor-machine-learning-model-sagemaker-studio](https://aws.amazon.com/tutorials/build-train-deploy-monitor-machine-learning-model-sagemaker-studio/) | Tutorial | 2h |
| SageMaker Experiments & Model Registry | [docs.aws.amazon.com/sagemaker/latest/dg/experiments](https://docs.aws.amazon.com/sagemaker/latest/dg/experiments.html) | Documentation | 1h |
| SHAP for ML Interpretability | [github.com/shap/shap](https://github.com/shap/shap) | Tool | 1h |
| SageMaker Clarify — Bias Detection | [docs.aws.amazon.com/sagemaker/latest/dg/clarify-configure-processing-jobs](https://docs.aws.amazon.com/sagemaker/latest/dg/clarify-configure-processing-jobs.html) | Documentation | 1h |

#### Hands-On Exercise (⏱ 8 hours)
- **Train three models on SageMaker:**
  ```python
  import sagemaker
  from sagemaker.xgboost import XGBoost
  from sagemaker.pytorch import PyTorch

  # Model 1: XGBoost (gradient boosted trees)
  xgb = XGBoost(
      entry_point='train_xgb.py',
      role=role,
      instance_count=1,
      instance_type='ml.m5.xlarge',
      framework_version='1.7-1',
      hyperparameters={
          'max_depth': 6,
          'eta': 0.1,
          'objective': 'binary:logistic',
          'scale_pos_weight': 3.0,  # handle class imbalance
          'eval_metric': 'auc',
          'num_round': 500
      }
  )
  xgb.fit({'train': train_s3, 'validation': val_s3})

  # Model 2: PyTorch neural network with attention
  # Model 3: Ensemble (average probabilities)
  ```
- **Evaluate with clinical genomics metrics:**
  - Precision, Recall, F1 (especially recall for pathogenic — missing a pathogenic variant is dangerous)
  - AUROC and AUPRC curves
  - Calibration plots (predicted probability vs observed frequency)
  - Per-gene performance breakdown
- **SHAP analysis for interpretability:**
  ```python
  import shap
  explainer = shap.TreeExplainer(model)
  shap_values = explainer.shap_values(X_test)
  shap.summary_plot(shap_values, X_test, feature_names=feature_names)
  ```
- **Deploy as SageMaker endpoint** for real-time variant scoring
- **Score all EDS VUS variants** and rank by predicted pathogenicity
- **Generate a report** of top 20 VUS variants predicted as likely pathogenic

#### Key Concepts to Master
- [ ] Why recall matters more than precision for pathogenic variant detection
- [ ] AUROC vs AUPRC: why AUPRC is better for imbalanced data
- [ ] SHAP values: which features drive each individual prediction
- [ ] Model calibration: ensuring predicted probabilities are meaningful
- [ ] Deployment considerations: real-time endpoint vs batch transform
- [ ] A/B testing models: SageMaker production variants
- [ ] Regulatory awareness: ML for variant interpretation is not FDA-cleared

---

## Phase 6: Advanced Topics

### Week 11 — Multi-Omics Integration & Graph ML

**🎯 Learning Goal:** Integrate genomics with proteomics, transcriptomics, and phenotype data. Build a knowledge graph of EDS genes, pathways, and variants using Amazon Neptune with Graph ML.

#### Resources
| Resource | URL | Type | Time |
|----------|-----|------|------|
| Amazon Neptune ML Documentation | [docs.aws.amazon.com/neptune/latest/userguide/machine-learning](https://docs.aws.amazon.com/neptune/latest/userguide/machine-learning.html) | Documentation | 2h |
| Neptune Analytics + GraphStorm | [aws.amazon.com/about-aws/whats-new/2025/06/amazon-neptune-analytics-integrates-graphstorm](https://aws.amazon.com/about-aws/whats-new/2025/06/amazon-neptune-analytics-integrates-graphstorm/) | Announcement | 1h |
| Bedrock Knowledge Bases + Neptune GraphRAG | [aws.amazon.com/blogs/machine-learning/announcing-general-availability-of-amazon-bedrock-knowledge-bases-graphrag-with-amazon-neptune-analytics](https://aws.amazon.com/blogs/machine-learning/announcing-general-availability-of-amazon-bedrock-knowledge-bases-graphrag-with-amazon-neptune-analytics/) | Blog | 2h |
| STRING Protein Interaction Database | [string-db.org](https://string-db.org/) | Database | 1h |
| Reactome Pathway Database | [reactome.org](https://reactome.org/) | Database | 1h |
| Deep Graph Library (DGL) | [dgl.ai](https://www.dgl.ai/) | Tool | 2h |
| AWS + NVIDIA Biology Knowledge Graph Hackathon | [aws.amazon.com/blogs/publicsector/bridging-ai-and-biology-inside-the-aws-and-nvidia-open-data-knowledge-graph-hackathon](https://aws.amazon.com/blogs/publicsector/bridging-ai-and-biology-inside-the-aws-and-nvidia-open-data-knowledge-graph-hackathon/) | Blog | 1h |

#### Hands-On Exercise (⏱ 8 hours)
- **Build an EDS knowledge graph in Neptune:**
  ```python
  # Define the schema
  # Nodes: Gene, Variant, Protein, Pathway, Phenotype (HPO), Disease, Publication
  # Edges: VARIANT_IN, ENCODES, INTERACTS_WITH, PARTICIPATES_IN, HAS_PHENOTYPE,
  #         CAUSES, CITED_IN, CO_EXPRESSED_WITH

  # Load EDS gene-gene interactions from STRING
  # Load pathway memberships from Reactome
  # Load variant-disease associations from ClinVar
  # Load phenotype annotations from HPO
  # Load protein structures from UniProt

  # Example Gremlin query: Find all genes interacting with COL5A1
  # that also have pathogenic variants in ClinVar
  g.V().has('Gene','name','COL5A1')
    .out('INTERACTS_WITH').has('Gene')
    .as('interactor')
    .out('HAS_VARIANT').has('Variant','clinvar_class','Pathogenic')
    .select('interactor').values('name')
  ```
- **Train a Graph Neural Network** with Neptune ML:
  - Node classification: predict whether a VUS variant is pathogenic based on graph neighborhood
  - Link prediction: discover missing gene-gene interactions
  - Use DGL + GNN for variant effect prediction using protein interaction context
- **Multi-omics feature integration:**
  - Add gene expression data (GTEx — connective tissue samples)
  - Add protein abundance data (Human Protein Atlas)
  - Explore: are EDS genes co-expressed in skin, tendon, and vascular tissues?

#### Key Concepts to Master
- [ ] Graph Neural Networks: message passing, node embeddings, GraphSAGE
- [ ] Neptune ML pipeline: data export → SageMaker training → Neptune inference
- [ ] GraphRAG: combining knowledge graphs with LLM retrieval
- [ ] Multi-omics: genomics + transcriptomics + proteomics + metabolomics
- [ ] Network biology: hub genes, modules, connectivity as a measure of gene importance
- [ ] Protein-protein interaction networks: why collagen gene neighborhoods matter for EDS

---

### Week 12 — Bedrock AgentCore & Federated Learning

**🎯 Learning Goal:** Build an AI agent for variant interpretation using Amazon Bedrock AgentCore. Explore federated learning concepts for privacy-preserving genomics research.

#### Resources
| Resource | URL | Type | Time |
|----------|-----|------|------|
| HealthOmics + Bedrock AgentCore Blog | [aws.amazon.com/blogs/machine-learning/accelerating-genomics-variant-interpretation-with-aws-healthomics-and-amazon-bedrock-agentcore](https://aws.amazon.com/blogs/machine-learning/accelerating-genomics-variant-interpretation-with-aws-healthomics-and-amazon-bedrock-agentcore/) | Blog + code | 3h |
| Bedrock AgentCore Documentation | [docs.aws.amazon.com/bedrock/latest/userguide/agentcore](https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html) | Documentation | 2h |
| Bedrock Knowledge Bases | [docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html) | Documentation | 1h |
| Flower — Federated Learning Framework | [flower.ai](https://flower.ai/) | Framework | 2h |
| NVIDIA FLARE for Healthcare | [github.com/NVIDIA/NVFlare](https://github.com/NVIDIA/NVFlare) | Framework | 1h |
| SageMaker Federated Learning | [docs.aws.amazon.com/sagemaker](https://docs.aws.amazon.com/sagemaker/latest/dg/distributed-training.html) | Documentation | 1h |

#### Hands-On Exercise (⏱ 8 hours)
- **Build an EDS Variant Interpretation Agent with Bedrock AgentCore:**
  ```python
  # Agent Architecture:
  # 1. Knowledge Base: ClinVar EDS variants, OMIM entries, GeneReviews,
  #    published EDS papers (RAG-indexed)
  # 2. Tools:
  #    - query_clinvar(gene, variant) → ClinVar classification
  #    - score_variant(chrom, pos, ref, alt) → CADD, REVEL, SpliceAI scores
  #    - query_gnomad(variant) → population frequency
  #    - query_eds_classifier(features) → your Week 10 model prediction
  #    - search_literature(query) → PubMed search
  # 3. Agent Instructions:
  #    "You are an EDS variant interpretation assistant. Given a genetic
  #     variant, retrieve all available evidence, apply ACMG/AMP criteria,
  #     and provide a structured interpretation report with confidence levels."

  import boto3
  bedrock_agent = boto3.client('bedrock-agent')

  # Create the agent with action groups for each tool
  # Connect to Neptune knowledge graph via Lambda functions
  # Test with known pathogenic and benign EDS variants
  ```
- **Natural language variant queries:**
  - "What is the clinical significance of COL5A1 c.2700+2T>C?"
  - "Are there any pathogenic splice variants in TNXB?"
  - "Compare the pathogenicity evidence for all COL3A1 glycine substitutions in exon 30"
- **Federated learning proof of concept:**
  - Set up a simple federated learning simulation with Flower
  - Simulate 3 hospitals each with local EDS variant data
  - Train a shared model without sharing raw patient data
  - Compare federated vs centralized model performance

#### Key Concepts to Master
- [ ] Bedrock AgentCore: agents, action groups, knowledge bases, guardrails
- [ ] RAG for genomics: indexing ClinVar, PubMed, OMIM for variant interpretation
- [ ] Tool-use agents: connecting AI reasoning with bioinformatics tools
- [ ] Federated learning: FedAvg algorithm, differential privacy, secure aggregation
- [ ] Privacy in genomics: re-identification risks, HIPAA, GDPR
- [ ] The future: foundation models for genomics (Evo, Enformer, AlphaGenome)

---

## 📚 Reading List

### Key 2025–2026 Papers

| Paper | Journal | Year | Focus | URL |
|-------|---------|------|-------|-----|
| Shirvani et al. — "Multi-System Genetic Architecture of hEDS: Integrating ML with Subject-Level Genomic Analysis" | MDPI Genes | 2026 | Machine learning for hEDS genetic architecture; identifies multi-system gene networks | [mdpi.com/2073-4425/17/2/211](https://www.mdpi.com/2073-4425/17/2/211) |
| aiDIVA — "Hybrid AI for Rare Disease Diagnostics Using Evidence-Based, ML and Language Models" | Nature npj Genomic Medicine | 2026 | Ensemble AI combining random forest + LLM for causal variant identification | [nature.com/articles/s41525-026-00611-x](https://www.nature.com/articles/s41525-026-00611-x) |
| GenPhenia — "Using Deep Neural Networks to Accelerate Rare-Disease Diagnosis" | Human Genomics | 2026 | DNN-based phenotype-driven gene prioritization for rare diseases | [springer.com/article/10.1186/s40246-026-01023-9](https://link.springer.com/article/10.1186/s40246-026-01023-9) |
| G.AI — "An AI-Driven Platform for Phenotype Standardization, Variant Interpretation and Structured Clinical Reporting" | J Translational Medicine | 2026 | End-to-end AI platform for rare disease genomic diagnosis | [springer.com/article/10.1186/s12967-026-08368-8](https://link.springer.com/article/10.1186/s12967-026-08368-8) |
| AWS Blog — "Accelerating Genomics Variant Interpretation with AWS HealthOmics and Amazon Bedrock AgentCore" | AWS ML Blog | Nov 2025 | HealthOmics + S3 Tables + Bedrock AgentCore for variant analysis | [aws.amazon.com/blogs/machine-learning/accelerating-genomics-variant-interpretation...](https://aws.amazon.com/blogs/machine-learning/accelerating-genomics-variant-interpretation-with-aws-healthomics-and-amazon-bedrock-agentcore/) |
| AlphaGenome — "Advancing Regulatory Variant Effect Prediction" | Nature | 2026 | DeepMind's multi-track regulatory genomics model (5,930 genomic signals) | [nature.com/articles/s41586-025-10014-0](https://www.nature.com/articles/s41586-025-10014-0) |
| Cheng et al. — "Accurate Proteome-Wide Missense Variant Effect Prediction with AlphaMissense" | Science | 2023 | AlphaFold-derived pathogenicity prediction for 71M missense variants | [science.org/doi/full/10.1126/science.adg7492](https://www.science.org/doi/full/10.1126/science.adg7492) |
| PubMind — "Literature-Based Genetic Variant Extraction Using LLMs" | Nature Communications | 2026 | LLM-powered variant extraction and functional annotation from literature | [nature.com/articles/s41467-026-76834-4](https://www.nature.com/articles/s41467-026-76834-4) |
| "Interpreting Human Genetic Variation at Atomic Resolution" | Nature Genetics | 2026 | Review of structure-based variant interpretation methods | [nature.com/articles/s41588-026-02763-z](https://www.nature.com/articles/s41588-026-02763-z) |
| gnomAD v4 — "Integrating 730,947 Exome Sequences" | Broad Institute | 2026 | Updated population frequency database with 5x more samples | [broadinstitute.org/publications/broad1376256](https://www.broadinstitute.org/publications/broad1376256) |

### Foundational Textbooks

| Book | Authors | Why Read It |
|------|---------|-------------|
| **Bioinformatics Algorithms: An Active Learning Approach** (3rd Ed) | Phillip Compeau & Pavel Pevzner | Gold standard for algorithmic bioinformatics; pairs with Rosalind.info problems. [compeau.cbd.cmu.edu](https://compeau.cbd.cmu.edu/online-education-projects/bioinformatics-algorithms-an-active-learning-approach/) |
| **Genome-Scale Algorithm Design** (2nd Ed, Cambridge 2023) | Mäkinen, Belazzougui, Cunial, Tomescu | Deep dive into FM-index, BWT, and the algorithms powering BWA/bowtie. Essential for pipeline understanding |
| **Bioinformatics Data Skills** | Vince Buffalo (O'Reilly) | Practical Unix, Python, R skills for genomic data. Best "getting things done" book for bioinformaticians |
| **Computational Intelligence for Genomics Data** | Elsevier, 2025 | ML/DL techniques specifically for genomic analysis and disease prediction. [elsevier.com](https://business.elsevier.com/computational-intelligence-for-genomics-data-9780443300806-html.html) |
| **Molecular Biology of the Cell** (7th Ed) | Alberts et al. | The definitive cell biology reference. Review chapters on extracellular matrix and connective tissue |
| **Thompson & Thompson Genetics in Medicine** (9th Ed) | Nussbaum, McInnes, Willard | Clinical genetics fundamentals: inheritance patterns, variant interpretation, genetic counseling |
| **Human Molecular Genetics** (5th Ed) | Strachan & Read | Deep molecular genetics with excellent chapters on disease mechanisms |

### YouTube Channels & Playlists

| Channel | Focus | URL |
|---------|-------|-----|
| **StatQuest (Josh Starmer)** | Statistics & ML fundamentals (PCA, random forests, neural networks) — exceptional visual explanations | [youtube.com/@statquest](https://www.youtube.com/@statquest) |
| **OMGenomics** | Bioinformatics tutorials, RNA-seq, variant analysis, getting started guides | [youtube.com/@OMGenomics](https://www.youtube.com/@OMGenomics) |
| **Broad Institute** | GATK workshops, genomics talks, clinical genomics seminars | [youtube.com/@broadinstitute](https://www.youtube.com/@broadinstitute) |
| **AWS Online Tech Talks** | HealthOmics demos, SageMaker for life sciences, cloud genomics architecture | [youtube.com/@AWSOnlineTechTalks](https://www.youtube.com/@AWSOnlineTechTalks) |
| **3Blue1Brown** | Neural networks, linear algebra, and calculus — brilliant visual math | [youtube.com/@3blue1brown](https://www.youtube.com/@3blue1brown) |
| **Bioinformatics Coach** | Step-by-step bioinformatics analysis protocols and tool tutorials | [youtube.com/@BioinformaticsCoach](https://www.youtube.com/@BioinformaticsCoach) |
| **NHGRI Genomics Education** | Human Genome Research Institute lectures and seminars | [youtube.com/@genome](https://www.youtube.com/@genome) |

### Podcasts

| Podcast | Focus | URL |
|---------|-------|-----|
| **Mendelspod** | Genomics and genomic medicine — interviews with field leaders | [mendelspod.com](https://www.mendelspod.com/) |
| **Genetics Unzipped** | Genetics Society podcast — genes, genomes, and DNA stories | [geneticsunzipped.com](https://geneticsunzipped.com/) |
| **TWIML AI Podcast** | ML/AI research (filter for healthcare/genomics episodes) | [twimlai.com](https://twimlai.com/) |
| **The Bioinformatics CRO Podcast** | Industry bioinformatics — NGS analysis, clinical genomics | Search on your podcast app |
| **Genomics and Health Disparities** | NHGRI podcast on equity in genomic medicine | Available on major platforms |

### GitHub Repositories to Star ⭐

| Repository | Description | URL |
|------------|-------------|-----|
| **google/deepvariant** | DL-based variant caller | [github.com/google/deepvariant](https://github.com/google/deepvariant) |
| **bwa-mem2/bwa-mem2** | Next-gen read aligner | [github.com/bwa-mem2/bwa-mem2](https://github.com/bwa-mem2/bwa-mem2) |
| **Illumina/SpliceAI** | DL splice variant predictor | [github.com/illumina/spliceAI](https://github.com/illumina/spliceAI) |
| **Ensembl/ensembl-vep** | Variant Effect Predictor | [github.com/Ensembl/ensembl-vep](https://github.com/Ensembl/ensembl-vep) |
| **broadinstitute/gatk** | Genome Analysis Toolkit | [github.com/broadinstitute/gatk](https://github.com/broadinstitute/gatk) |
| **aws-samples/amazon-omics-tutorials** | HealthOmics Jupyter notebooks | [github.com/aws-samples/amazon-omics-tutorials](https://github.com/aws-samples/amazon-omics-tutorials/) |
| **aws-samples/amazon-omics-end-to-end-genomics** | Full HealthOmics pipeline | [github.com/aws-samples/amazon-omics-end-to-end-genomics](https://github.com/aws-samples/amazon-omics-end-to-end-genomics) |
| **awslabs/genomics-tertiary-analysis-and-data-lakes** | Multi-omics data lake on AWS | [github.com/awslabs/genomics-tertiary-analysis-and-data-lakes-using-aws-glue-and-amazon-athena](https://github.com/awslabs/genomics-tertiary-analysis-and-data-lakes-using-aws-glue-and-amazon-athena) |
| **shap/shap** | ML interpretability | [github.com/shap/shap](https://github.com/shap/shap) |
| **dmlc/dgl** | Deep Graph Library for GNN | [github.com/dmlc/dgl](https://github.com/dmlc/dgl) |
| **adap/flower** | Federated learning framework | [github.com/adap/flower](https://github.com/adap/flower) |
| **pcingola/SnpEff** | Variant annotation & effect prediction | [github.com/pcingola/SnpEff](https://github.com/pcingola/SnpEff) |
| **Kuanhao-Chao/OpenSpliceAI** | Open-source SpliceAI reimplementation (PyTorch) | [github.com/Kuanhao-Chao/OpenSpliceAI](https://github.com/Kuanhao-Chao/OpenSpliceAI) |

---

## 🗄️ Datasets & Databases

### Core Databases

| Database | Description | URL | Use in EDS Research |
|----------|-------------|-----|---------------------|
| **ClinVar** | Clinical significance of genetic variants | [ncbi.nlm.nih.gov/clinvar](https://www.ncbi.nlm.nih.gov/clinvar/) | Primary source for labeled EDS variants (P/LP/VUS/B/LB); model training data |
| **gnomAD v4** | Population allele frequencies (730K+ exomes) | [gnomad.broadinstitute.org](https://gnomad.broadinstitute.org/) | Filter rare variants; population-specific frequency for EDS genes |
| **dbSNP** | Database of single nucleotide polymorphisms | [ncbi.nlm.nih.gov/snp](https://www.ncbi.nlm.nih.gov/snp/) | Variant identifiers (rsIDs); links to other databases |
| **OMIM** | Online Mendelian Inheritance in Man | [omim.org](https://www.omim.org/) | Gene-disease relationships; EDS subtype catalog |
| **HPO** | Human Phenotype Ontology | [hpo.jax.org](https://hpo.jax.org/) | Standardized phenotype terms for EDS features (hypermobility, skin elasticity, etc.) |
| **UniProt** | Protein sequence & function | [uniprot.org](https://www.uniprot.org/) | Collagen protein domains, post-translational modifications, functional annotations |
| **STRING** | Protein-protein interactions | [string-db.org](https://string-db.org/) | Gene interaction networks for collagen genes; input for graph ML |
| **Reactome** | Biological pathways | [reactome.org](https://reactome.org/) | Collagen biosynthesis pathways; ECM organization |
| **DECIPHER** | Genomic variant clinical interpretation | [deciphergenomics.org](https://deciphergenomics.org/) | Clinical-grade variant data with phenotype links |
| **ClinGen** | Clinical Genome Resource | [clinicalgenome.org](https://www.clinicalgenome.org/) | Gene-disease validity curation; variant interpretation expert panels |
| **InterPro** | Protein domain classification | [ebi.ac.uk/interpro](https://www.ebi.ac.uk/interpro/) | Map variants to functional protein domains |
| **GTEx** | Genotype-Tissue Expression | [gtexportal.org](https://gtexportal.org/) | Tissue-specific expression of EDS genes |

### AWS Registry of Open Data — Genomics Datasets

| Dataset | Description | S3 Location / URL |
|---------|-------------|-------------------|
| **1000 Genomes Project** | 3,202 whole genomes, population genetics | [registry.opendata.aws/1000-genomes](https://registry.opendata.aws/1000-genomes/) — `s3://1000genomes/` |
| **1000 Genomes DRAGEN Reanalysis** | Re-analyzed with DRAGEN v3.5–4.4 (2026) | [aws.amazon.com/blogs/publicsector/the-1000-genomes-project-reanalyzed](https://aws.amazon.com/blogs/publicsector/the-1000-genomes-project-reanalyzed-a-new-analytical-baseline-for-human-genomics/) |
| **ClinVar on Open Data** | ClinVar variant summaries in cloud-queryable format | [registry.opendata.aws](https://registry.opendata.aws/) — search "ClinVar" |
| **gnomAD on Open Data** | Population frequency data | `s3://gnomad-public-us-east-1/` |
| **AWS iGenomes** | Pre-built reference genomes (GRCh37, GRCh38) with indices | [registry.opendata.aws/aws-igenomes](https://registry.opendata.aws/aws-igenomes/) |
| **ENCODE** | Encyclopedia of DNA Elements — functional annotations | [registry.opendata.aws](https://registry.opendata.aws/) — search "ENCODE" |
| **AnVIL Genomic Datasets** | Broad Institute managed genomic datasets (free on AWS since 2026) | [news.ucsc.edu/2026/04/anvil-makes-genomic-datasets-free](https://news.ucsc.edu/2026/04/anvil-makes-genomic-datasets-free/) |

---

## 🎯 EDS Gene Reference Panel

These are the primary genes to focus on throughout the study plan:

| Gene | Protein | EDS Subtype | Inheritance | Key Variant Types |
|------|---------|-------------|-------------|-------------------|
| **COL5A1** | Type V Collagen α1 | Classical (cEDS) | AD | Null alleles, splice, missense |
| **COL5A2** | Type V Collagen α2 | Classical (cEDS) | AD | Missense (Gly substitutions) |
| **COL3A1** | Type III Collagen α1 | Vascular (vEDS) | AD | Gly substitutions in triple helix — **life-threatening** |
| **COL1A1** | Type I Collagen α1 | Arthrochalasia (aEDS) | AD | Exon 6 skipping |
| **COL1A2** | Type I Collagen α2 | Arthrochalasia (aEDS), Cardiac-valvular | AD / AR | Exon 6 skipping; biallelic null |
| **TNXB** | Tenascin-X | Classical-like (clEDS) | AR | Null alleles |
| **COL12A1** | Type XII Collagen α1 | Myopathic (mEDS) | AD / AR | Missense, null |
| **ADAMTS2** | Procollagen I N-proteinase | Dermatosparaxis (dEDS) | AR | Null alleles |
| **PLOD1** | Lysyl hydroxylase 1 | Kyphoscoliotic (kEDS) | AR | Null, missense |
| **FKBP14** | FKBP prolyl isomerase 14 | Kyphoscoliotic (kEDS) | AR | Splice, missense |
| **B4GALT7** | Galactosyltransferase I | Spondylodysplastic (spEDS) | AR | Missense |
| **B3GALT6** | Galactosyltransferase II | Spondylodysplastic (spEDS) | AR | Null, missense |
| **SLC39A13** | Zinc transporter ZIP13 | Spondylodysplastic (spEDS) | AR | Missense |
| **CHST14** | Dermatan 4-sulfotransferase | Musculocontractural (mcEDS) | AR | Null alleles |
| **DSE** | Dermatan sulfate epimerase | Musculocontractural (mcEDS) | AR | Null, missense |
| **AEBP1** | Aortic carboxypeptidase-like protein | Classical-like (clEDS) | AR | Null alleles |
| **Unknown** | — | Hypermobile (hEDS) | Unknown | **No gene identified — this is the research frontier** |

---

## ✅ Weekly Progress Tracker

| Week | Theme | Status | Hours Logged | Notes |
|------|-------|--------|-------------|-------|
| 1 | Bioinformatics Data & Formats | ⬜ Not started | — | |
| 2 | Galaxy & Sequencing Technologies | ⬜ Not started | — | |
| 3 | Alignment & Variant Calling | ⬜ Not started | — | |
| 4 | Annotation & EDS Gene Focus | ⬜ Not started | — | |
| 5 | AWS HealthOmics Setup | ⬜ Not started | — | |
| 6 | Cloud-Scale Genomics | ⬜ Not started | — | |
| 7 | Pathogenicity Prediction Tools | ⬜ Not started | — | |
| 8 | Advanced Interpretation & EDS Analysis | ⬜ Not started | — | |
| 9 | Data Engineering & Feature Design | ⬜ Not started | — | |
| 10 | Model Training & Deployment | ⬜ Not started | — | |
| 11 | Multi-Omics & Graph ML | ⬜ Not started | — | |
| 12 | Bedrock AgentCore & Federated Learning | ⬜ Not started | — | |

---

## 🏗️ Capstone Project Idea

**"EDS Variant Oracle"** — An end-to-end AI system for EDS variant interpretation:

1. **Input:** Raw FASTQ files or VCF from a patient suspected of EDS
2. **Pipeline:** HealthOmics workflow → alignment → variant calling → annotation
3. **AI Layer:** Your trained EDS classifier + AlphaMissense + CADD + SpliceAI scores
4. **Knowledge Graph:** Neptune graph connecting variants to genes, pathways, phenotypes, and literature
5. **Agent:** Bedrock AgentCore-powered conversational interface for clinicians
6. **Output:** Structured clinical report with ACMG-style classification, evidence summary, and recommended follow-up

This would combine everything learned across all 12 weeks into a portfolio-worthy project that demonstrates real translational genomics capability.

---

*Last updated: September 2026 | Created for personal learning — not medical advice*
