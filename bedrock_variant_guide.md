# Building an LLM-Powered Genomic Variant Interpretation Agent

## AWS Bedrock AgentCore + HealthOmics + Strands SDK

**Target audience:** AWS Solutions Architects with deep AWS fluency, new to clinical genomics & AI-assisted variant interpretation.

**Use case:** Automated interpretation of Ehlers-Danlos Syndrome (EDS) variants using ACMG/AMP classification standards, powered by agentic AI.

> **Key reference:** This guide extends the architecture described in the AWS blog post *"Accelerating genomics variant interpretation with AWS HealthOmics and Amazon Bedrock AgentCore"* (Nov 2025, by Sandanaraj, Lee & Poonawala) with a focus on clinical-grade ACMG interpretation, EDS-specific rules, and a complete tool chain for rare disease research.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Setting Up HealthOmics Annotation Store](#2-setting-up-healthomics-annotation-store)
3. [Building the Bedrock Agent with Strands SDK](#3-building-the-bedrock-agent-with-strands-sdk)
4. [ACMG Rules Engine](#4-acmg-rules-engine)
5. [Example Walkthrough: COL3A1 c.1859G>A](#5-example-walkthrough-col3a1-c1859ga)
6. [RAG Pattern for Genomics Literature](#6-rag-pattern-for-genomics-literature)
7. [Deployment & Cost Estimation](#7-deployment--cost-estimation)

---

## 1. Architecture Overview

### System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        GENOMIC VARIANT INTERPRETATION AGENT                 │
│                                                                             │
│  ┌─────────────┐    ┌──────────────────────────────────────────────────┐    │
│  │             │    │         Amazon Bedrock AgentCore Runtime         │    │
│  │   User      │    │  ┌────────────────────────────────────────────┐  │    │
│  │   Query     │───▶│  │  Strands Orchestrator Agent                │  │    │
│  │             │    │  │  (Claude Sonnet 4 / Haiku)                 │  │    │
│  │ "Interpret  │    │  │                                            │  │    │
│  │  COL3A1     │    │  │  System Prompt: Genomic Variant            │  │    │
│  │  c.1859G>A" │    │  │  Interpretation Specialist                 │  │    │
│  │             │    │  │                                            │  │    │
│  └─────────────┘    │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐  │  │    │
│                     │  │  │ Reasoning │ │   Tool   │ │Synthesis │  │  │    │
│                     │  │  │   Loop    │▶│  Calls   │▶│  & Report│  │  │    │
│                     │  │  └──────────┘ └────┬─────┘ └──────────┘  │  │    │
│                     │  └────────────────────┼─────────────────────┘  │    │
│                     └───────────────────────┼────────────────────────┘    │
│                                             │                             │
│                    ┌────────────────────────┼────────────────────────┐    │
│                    │              TOOL LAYER │                        │    │
│                    │                        ▼                        │    │
│         ┌──────────┴──────────┬─────────────┴──────┬────────────┐   │    │
│         ▼                     ▼                    ▼            ▼   │    │
│  ┌──────────────┐  ┌───────────────────┐  ┌────────────┐ ┌────────┐│    │
│  │query_annota- │  │   query_vep       │  │search_pub- │ │apply_  ││    │
│  │tion_store    │  │                   │  │med         │ │acmg_   ││    │
│  │              │  │  Ensembl VEP      │  │            │ │criteria││    │
│  │ HealthOmics  │  │  REST API         │  │ NCBI       │ │        ││    │
│  │ + S3 Tables  │  │  (+ CADD, REVEL,  │  │ E-Utils    │ │Python  ││    │
│  │ + Athena     │  │   SpliceAI,       │  │ API        │ │Rules   ││    │
│  │              │  │   AlphaMissense)  │  │            │ │Engine  ││    │
│  └──────┬───────┘  └────────┬──────────┘  └─────┬──────┘ └───┬────┘│    │
│         │                   │                    │            │     │    │
│         ▼                   ▼                    ▼            ▼     │    │
│  ┌──────────────┐  ┌───────────────────┐  ┌────────────┐ ┌────────┐│    │
│  │ ClinVar      │  │ Functional        │  │ Literature │ │ ACMG   ││    │
│  │ gnomAD       │  │ Predictions       │  │ Evidence   │ │ 5-tier ││    │
│  │ Annotations  │  │ Consequence       │  │ Citations  │ │ Class  ││    │
│  └──────────────┘  └───────────────────┘  └────────────┘ └────────┘│    │
│                                                                     │    │
│                    ┌────────────────────────────────────────────┐    │    │
│                    │              RAG LAYER (Optional)          │    │    │
│                    │                                            │    │    │
│                    │  Bedrock Knowledge Base                    │    │    │
│                    │  ├── EDS research papers (PMC/PubMed)      │    │    │
│                    │  ├── Gene-specific ACMG specs              │    │    │
│                    │  └── Clinical practice guidelines          │    │    │
│                    │                                            │    │    │
│                    │  OpenSearch Serverless (vector store)       │    │    │
│                    │  Titan Embeddings v2 (1024-dim)            │    │    │
│                    └────────────────────────────────────────────┘    │    │
│                                                                     │    │
│  ┌─────────────────────────────────────────────────────────────┐    │    │
│  │                    FINAL OUTPUT                              │    │    │
│  │                                                              │    │    │
│  │  ┌──────────────────────────────────────────────────────┐   │    │    │
│  │  │  Variant Interpretation Report                        │   │    │    │
│  │  │  ─────────────────────────────────────────            │   │    │    │
│  │  │  Classification: Likely Pathogenic (Class 4)          │   │    │    │
│  │  │  ACMG Criteria: PM1 + PM2 + PP3 + PM5                │   │    │    │
│  │  │  Gene: COL3A1 | Condition: vEDS (OMIM 130050)        │   │    │    │
│  │  │  Evidence Summary: [structured report]                │   │    │    │
│  │  │  Literature: [cited references]                       │   │    │    │
│  │  └──────────────────────────────────────────────────────┘   │    │    │
│  └─────────────────────────────────────────────────────────────┘    │    │
└─────────────────────────────────────────────────────────────────────────────┘

```

### Data Flow

1. **User submits a variant** in HGVS notation (e.g., `COL3A1:c.1859G>A`)
2. **Strands Agent** on AgentCore Runtime receives the query, enters its agentic reasoning loop
3. **Tool calls execute in parallel where possible:**- `query_annotation_store` → Athena SQL against S3 Tables (ClinVar clinical significance, gnomAD allele frequencies)

- `query_vep` → Ensembl VEP REST API (consequence type, protein effect, in-silico predictions: CADD, REVEL, SpliceAI, AlphaMissense)
- `search_pubmed` → NCBI E-utilities (relevant literature for gene + variant)

1. **Evidence aggregation:** Agent collects all tool results
2. `apply_acmg_criteria` → Python rules engine evaluates ACMG/AMP criteria against collected evidence
3. **LLM synthesis** → Agent generates a structured clinical interpretation report with classification, evidence summary, and citations

### Component Responsibilities

| Component | Role | AWS Service |
| --- | --- | --- |
| **Orchestrator** | Agentic loop: reason → select tool → execute → synthesize | Bedrock AgentCore + Strands SDK |
| **Annotation Store** | Pre-indexed ClinVar + gnomAD variant data | HealthOmics → S3 Tables → Athena |
| **VEP API** | Real-time functional annotation + in-silico predictions | External (Ensembl REST) |
| **PubMed Search** | Literature evidence retrieval | External (NCBI E-utilities) |
| **ACMG Engine** | Deterministic rules-based variant classification | Lambda / inline function |
| **Knowledge Base** | RAG over EDS literature corpus | Bedrock KB + OpenSearch Serverless |
| **LLM** | Reasoning, synthesis, report generation | Claude Sonnet 4 via Bedrock |

---

## 2. Setting Up HealthOmics Annotation Store

> **Important note (Sep 2025+):** AWS HealthOmics variant stores and annotation stores are no longer open to **new** customers as of Nov 7, 2025. Existing customers can continue using the service. The AWS blog post's recommended architecture uses **HealthOmics Workflows + S3 Tables + Athena** as the data layer — this is the forward-looking pattern. The steps below cover both approaches.

### 2a. Modern Pattern: HealthOmics Workflows → S3 Tables → Athena

This is the architecture from the AWS blog post. VCF files are annotated via HealthOmics VEP workflows, transformed to Apache Iceberg format in S3 Tables, and queried via Athena.

Step 1: Create an S3 Bucket for Raw VCFs

```bash
# Create bucket for raw VCF files
aws s3 mb s3://genomics-variant-store-${AWS_ACCOUNT_ID} --region us-east-1

# Upload your VCF files
aws s3 cp sample.vcf.gz \
  s3://genomics-variant-store-${AWS_ACCOUNT_ID}/raw-vcf/

```

Step 2: Create a HealthOmics Reference Store

```bash
# Create reference store for GRCh38
aws omics create-reference-store \
  --name "grch38-reference" \
  --description "GRCh38 human reference genome"

# Import the reference genome
aws omics start-reference-import-job \
  --reference-store-id <reference-store-id> \
  --sources '[{
    "sourceFile": "s3://genomics-variant-store-${AWS_ACCOUNT_ID}/reference/GRCh38.fa",
    "name": "GRCh38",
    "description": "Human reference genome GRCh38"
  }]' \
  --role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/OmicsImportRole

```

Step 3: Create a VEP Annotation Workflow

```bash
# Create a HealthOmics private workflow for VEP annotation
aws omics create-workflow \
  --name "vep-annotation-workflow" \
  --description "VEP annotation pipeline for VCF files" \
  --engine WDL \
  --definition-zip fileb://vep-workflow.zip \
  --parameter-template '{
    "input_vcf": {"description": "Input VCF file S3 URI"},
    "reference_genome": {"description": "Reference genome version", "optional": true},
    "output_bucket": {"description": "S3 bucket for annotated output"}
  }'

```

Step 4: Run VEP Annotation

```bash
# Start a workflow run
aws omics start-run \
  --workflow-id <workflow-id> \
  --name "vep-annotation-run-001" \
  --role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/OmicsWorkflowRole \
  --output-uri s3://genomics-variant-store-${AWS_ACCOUNT_ID}/workflow-output/ \
  --parameters '{
    "input_vcf": "s3://genomics-variant-store-${AWS_ACCOUNT_ID}/raw-vcf/sample.vcf.gz",
    "reference_genome": "GRCh38",
    "output_bucket": "s3://genomics-variant-store-${AWS_ACCOUNT_ID}/vep-output/"
  }'

```

Step 5: Create S3 Tables (Iceberg Format) via PyIceberg

```python
"""
Transform VEP-annotated VCF + ClinVar to Iceberg tables in S3 Tables.
This runs on AWS Batch (Fargate) triggered by EventBridge on workflow completion.
"""
from pyiceberg.catalog import load_catalog
from pyiceberg.schema import Schema
from pyiceberg.types import (
    NestedField, StringType, IntegerType, FloatType, LongType
)
import pyarrow as pa
import pyarrow.parquet as pq

# Connect to the S3 Tables Iceberg REST endpoint
catalog = load_catalog(
    "s3tables",
    **{
        "type": "rest",
        "uri": f"https://s3tables.{region}.amazonaws.com/iceberg",
        "s3tables.table-bucket-arn": f"arn:aws:s3tables:{region}:{account_id}:bucket/genomics-tables",
        "rest.signing-name": "s3tables",
        "rest.signing-region": region,
    }
)

# Define the variant annotations schema
variant_schema = Schema(
    NestedField(1, "sample_id", StringType(), required=True),
    NestedField(2, "chrom", StringType(), required=True),
    NestedField(3, "pos", LongType(), required=True),
    NestedField(4, "ref", StringType(), required=True),
    NestedField(5, "alt", StringType(), required=True),
    NestedField(6, "gene_symbol", StringType()),
    NestedField(7, "consequence", StringType()),
    NestedField(8, "impact", StringType()),         # HIGH/MODERATE/LOW/MODIFIER
    NestedField(9, "hgvsc", StringType()),           # HGVS coding
    NestedField(10, "hgvsp", StringType()),          # HGVS protein
    NestedField(11, "cadd_phred", FloatType()),
    NestedField(12, "revel_score", FloatType()),
    NestedField(13, "spliceai_max", FloatType()),
    NestedField(14, "alphamissense", FloatType()),
    NestedField(15, "gnomad_af", FloatType()),       # gnomAD allele frequency
    NestedField(16, "clinvar_clnsig", StringType()), # ClinVar clinical significance
    NestedField(17, "clinvar_clnrevstat", StringType()),
    NestedField(18, "clinvar_clndn", StringType()),  # ClinVar disease name
    NestedField(19, "transcript_id", StringType()),
    NestedField(20, "protein_position", IntegerType()),
)

# Create the table with sample+chromosome partitioning
catalog.create_table(
    identifier="genomics_db.variant_annotations",
    schema=variant_schema,
    partition_spec=[("sample_id",), ("chrom",)],
)

print("✓ Iceberg table 'variant_annotations' created in S3 Tables")

```

Step 6: Query via Athena

```sql
-- Register the S3 Tables catalog in Athena (one-time setup)
-- Then query annotated variants:

SELECT
    sample_id,
    gene_symbol,
    hgvsc,
    hgvsp,
    consequence,
    impact,
    clinvar_clnsig,
    gnomad_af,
    cadd_phred,
    revel_score
FROM genomics_db.variant_annotations
WHERE gene_symbol = 'COL3A1'
  AND clinvar_clnsig LIKE '%athogenic%'
ORDER BY cadd_phred DESC;

```

### 2b. Legacy Pattern: HealthOmics Annotation Store (Existing Customers)

For existing HealthOmics annotation store customers:

```bash
# Create annotation store for ClinVar
aws omics create-annotation-store \
  --name "clinvar-annotations" \
  --store-format VCF \
  --reference referenceArn=arn:aws:omics:us-east-1:${AWS_ACCOUNT_ID}:referenceStore/<ref-store-id>/reference/<ref-id> \
  --description "ClinVar variant annotations" \
  --tags Project=GenomicAgent,DataSource=ClinVar

# Import ClinVar VCF
aws omics start-annotation-import-job \
  --destination-name "clinvar-annotations" \
  --items source=s3://genomics-variant-store-${AWS_ACCOUNT_ID}/clinvar/clinvar_20250901.vcf.gz \
  --role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/OmicsAnnotationImportRole \
  --format-options '{"formatToHeader": {"CHROM": "CHROM", "POS": "POS", "REF": "REF", "ALT": "ALT"}}'

# Create annotation store for gnomAD
aws omics create-annotation-store \
  --name "gnomad-frequencies" \
  --store-format VCF \
  --reference referenceArn=arn:aws:omics:us-east-1:${AWS_ACCOUNT_ID}:referenceStore/<ref-store-id>/reference/<ref-id> \
  --description "gnomAD population allele frequencies"

# Import gnomAD (by chromosome for parallelism)
for CHR in $(seq 1 22) X Y; do
  aws omics start-annotation-import-job \
    --destination-name "gnomad-frequencies" \
    --items source=s3://gnomad-public-data/release/4.0/vcf/genomes/gnomad.genomes.v4.0.sites.chr${CHR}.vcf.bgz \
    --role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/OmicsAnnotationImportRole
done

```

### IAM Roles and Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "HealthOmicsWorkflowAccess",
      "Effect": "Allow",
      "Action": [
        "omics:StartRun",
        "omics:GetRun",
        "omics:ListRuns",
        "omics:CreateWorkflow",
        "omics:StartAnnotationImportJob",
        "omics:GetAnnotationImportJob",
        "omics:GetAnnotationStore",
        "omics:ListAnnotationStores"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3TablesAccess",
      "Effect": "Allow",
      "Action": [
        "s3tables:CreateTable",
        "s3tables:GetTable",
        "s3tables:PutTableData",
        "s3tables:GetTableData"
      ],
      "Resource": "arn:aws:s3tables:*:${AWS_ACCOUNT_ID}:bucket/genomics-tables/*"
    },
    {
      "Sid": "AthenaQueryAccess",
      "Effect": "Allow",
      "Action": [
        "athena:StartQueryExecution",
        "athena:GetQueryExecution",
        "athena:GetQueryResults"
      ],
      "Resource": "*"
    },
    {
      "Sid": "GlueCatalogAccess",
      "Effect": "Allow",
      "Action": [
        "glue:GetDatabase",
        "glue:GetTable",
        "glue:GetPartitions"
      ],
      "Resource": [
        "arn:aws:glue:*:${AWS_ACCOUNT_ID}:catalog",
        "arn:aws:glue:*:${AWS_ACCOUNT_ID}:database/genomics_db",
        "arn:aws:glue:*:${AWS_ACCOUNT_ID}:table/genomics_db/*"
      ]
    },
    {
      "Sid": "S3DataAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::genomics-variant-store-${AWS_ACCOUNT_ID}",
        "arn:aws:s3:::genomics-variant-store-${AWS_ACCOUNT_ID}/*"
      ]
    }
  ]
}

```

---

## 3. Building the Bedrock Agent with Strands SDK

### Agent System Prompt

This prompt instructs the LLM on its role, domain expertise, and interpretation methodology:

```python
VARIANT_INTERPRETER_SYSTEM_PROMPT = """You are a clinical genomics variant interpretation specialist.
Your role is to analyze genetic variants using the ACMG/AMP framework (Richards et al., 2015)
and provide evidence-based classifications.

## Your Expertise
- ACMG/AMP 5-tier variant classification (Pathogenic → Benign)
- Connective tissue disorder genetics, especially Ehlers-Danlos Syndrome (all subtypes)
- COL3A1, COL5A1, COL5A2, COL1A1, COL1A2, COL12A1, TNXB, PLOD1, ADAMTS2, FKBP14
- In-silico pathogenicity predictors: CADD, REVEL, SpliceAI, AlphaMissense, SIFT, PolyPhen
- Population frequency databases: gnomAD, ExAC, 1000 Genomes
- Clinical databases: ClinVar, LOVD, OMIM, HGMD

## Interpretation Protocol
For each variant you MUST:
1. **Identify the variant**: Parse HGVS notation, identify gene, transcript, consequence
2. **Gather evidence**: Call tools to retrieve:
   - ClinVar/gnomAD data (query_annotation_store)
   - VEP functional annotation with in-silico scores (query_vep)
   - Published literature (search_pubmed)
3. **Apply ACMG criteria**: Use apply_acmg_criteria with ALL collected evidence
4. **Synthesize**: Generate a structured report with:
   - Variant identification (gene, transcript, protein change, genomic coordinates)
   - ACMG classification with specific criteria cited
   - Evidence summary organized by: population data, computational predictions,
     functional data, segregation, de novo, clinical data
   - Literature citations
   - Limitations and caveats

## EDS-Specific Rules
For collagen genes (COL3A1, COL5A1, COL1A1, etc.):
- Glycine substitutions in Gly-X-Y repeats of the triple helical domain are typically
  pathogenic (PM1 at minimum, often PVS1-equivalent for vascular EDS)
- Position within the triple helix matters: C-terminal substitutions in COL3A1
  are generally more severe than N-terminal
- Haploinsufficiency variants (PTC, frameshift) in COL3A1 cause a milder vEDS phenotype
  than dominant-negative glycine substitutions
- Splice variants affecting in-frame exon skipping produce dominant-negative effects

## Output Format
Always produce a structured report with these sections:
### Variant Identification
### ACMG Classification
### Evidence Summary
### Clinical Significance
### Limitations
### References

CRITICAL: Never classify a variant without tool-derived evidence. If a tool fails,
acknowledge the missing evidence and note its impact on classification confidence."""

```

### Tool Definitions with Strands @tool Decorator

Tool 1: query_annotation_store

```python
"""Query the HealthOmics annotation store / S3 Tables via Athena."""

import json
import time
import boto3
from strands import tool

athena_client = boto3.client("athena", region_name="us-east-1")

ATHENA_OUTPUT = "s3://genomics-variant-store-{account_id}/athena-results/"
ATHENA_DATABASE = "genomics_db"


@tool
def query_annotation_store(
    chrom: str,
    pos: int,
    ref: str,
    alt: str,
    gene: str = "",
) -> dict:
    """Query ClinVar and gnomAD annotations for a specific variant from the
    HealthOmics annotation store backed by S3 Tables and Athena.

    Args:
        chrom: Chromosome (e.g., "2", "X"). Do not include "chr" prefix.
        pos: Genomic position (GRCh38 coordinates).
        ref: Reference allele.
        alt: Alternate allele.
        gene: Gene symbol for broader gene-level queries (optional).

    Returns:
        Dictionary containing ClinVar clinical significance, review status,
        associated conditions, gnomAD allele frequencies across populations,
        and variant identifiers.
    """
    # Build SQL query for exact variant match
    sql = f"""
    SELECT
        chrom, pos, ref, alt,
        gene_symbol,
        hgvsc, hgvsp,
        consequence, impact,
        clinvar_clnsig,
        clinvar_clnrevstat,
        clinvar_clndn,
        gnomad_af,
        cadd_phred,
        revel_score,
        spliceai_max,
        alphamissense
    FROM {ATHENA_DATABASE}.variant_annotations
    WHERE chrom = '{chrom}'
      AND pos = {pos}
      AND ref = '{ref}'
      AND alt = '{alt}'
    """

    if gene:
        sql += f"  AND gene_symbol = '{gene}'\n"

    sql += "LIMIT 50;"

    # Execute Athena query
    response = athena_client.start_query_execution(
        QueryString=sql,
        QueryExecutionContext={"Database": ATHENA_DATABASE},
        ResultConfiguration={"OutputLocation": ATHENA_OUTPUT},
    )
    query_id = response["QueryExecutionId"]

    # Poll for completion (production: use Step Functions or EventBridge)
    while True:
        status = athena_client.get_query_execution(QueryExecutionId=query_id)
        state = status["QueryExecution"]["Status"]["State"]
        if state in ("SUCCEEDED", "FAILED", "CANCELLED"):
            break
        time.sleep(1)

    if state != "SUCCEEDED":
        return {
            "error": f"Athena query failed: {state}",
            "query": sql
        }

    # Fetch results
    results = athena_client.get_query_results(QueryExecutionId=query_id)
    rows = results["ResultSet"]["Rows"]

    if len(rows) <= 1:
        return {
            "found": False,
            "message": f"No annotations found for {chrom}:{pos} {ref}>{alt}",
            "query_used": sql,
        }

    headers = [col["VarCharValue"] for col in rows[0]["Data"]]
    annotations = []
    for row in rows[1:]:
        values = [col.get("VarCharValue", "") for col in row["Data"]]
        annotations.append(dict(zip(headers, values)))

    return {
        "found": True,
        "variant": f"{chrom}:{pos} {ref}>{alt}",
        "annotations": annotations,
        "result_count": len(annotations),
    }

```

Tool 2: query_vep

```python
"""Query the Ensembl Variant Effect Predictor REST API."""

import requests
from strands import tool

VEP_BASE = "https://rest.ensembl.org"


@tool
def query_vep(
    hgvs_notation: str,
    include_cadd: bool = True,
    include_revel: bool = True,
    include_spliceai: bool = True,
    include_alphamissense: bool = True,
) -> dict:
    """Query the Ensembl Variant Effect Predictor (VEP) REST API for comprehensive
    variant annotation including functional consequences and in-silico predictions.

    The VEP API provides: consequence type (missense, frameshift, splice, etc.),
    protein impact predictions, conservation scores, CADD/REVEL/SpliceAI/AlphaMissense
    pathogenicity scores, and known co-located variants.

    Args:
        hgvs_notation: Variant in HGVS notation. Supports coding (c.), genomic (g.),
            or protein (p.) notation with transcript/gene/chromosome reference.
            Examples: "ENST00000304636:c.1859G>A", "COL3A1:c.1859G>A",
            "2:g.188974589C>T"
        include_cadd: Include CADD deleteriousness scores (default True).
        include_revel: Include REVEL pathogenicity scores (default True).
        include_spliceai: Include SpliceAI splice prediction scores (default True).
        include_alphamissense: Include AlphaMissense pathogenicity scores (default True).

    Returns:
        Dictionary with VEP annotation results including consequence, impact,
        amino acid change, protein position, in-silico scores, overlapping domains,
        and co-located known variants.
    """
    # Build query parameters
    params = {
        "content-type": "application/json",
        "hgvs": 1,          # Include HGVS nomenclature
        "canonical": 1,     # Flag canonical transcript
        "mane": 1,          # Include MANE Select
        "protein": 1,       # Include protein identifiers
        "domains": 1,       # Include overlapping protein domains
        "numbers": 1,       # Include exon/intron numbers
        "variant_class": 1, # Include variant class (SNV, etc.)
    }

    if include_cadd:
        params["CADD"] = 1
    if include_revel:
        params["REVEL"] = 1
    if include_spliceai:
        params["SpliceAI"] = 2  # MANE transcript raw scores
    if include_alphamissense:
        params["AlphaMissense"] = 1

    # URL-encode the HGVS notation for the REST endpoint
    url = f"{VEP_BASE}/vep/human/hgvs/{requests.utils.quote(hgvs_notation)}"

    try:
        response = requests.get(url, params=params, timeout=30)
        response.raise_for_status()
        vep_data = response.json()
    except requests.exceptions.HTTPError as e:
        return {
            "error": f"VEP API error: {e.response.status_code} - {e.response.text}",
            "hgvs_queried": hgvs_notation,
        }
    except requests.exceptions.RequestException as e:
        return {
            "error": f"VEP API connection error: {str(e)}",
            "hgvs_queried": hgvs_notation,
        }

    if not vep_data:
        return {
            "found": False,
            "message": f"No VEP results for {hgvs_notation}",
        }

    # Extract the most relevant transcript consequence
    result = vep_data[0]
    transcript_consequences = result.get("transcript_consequences", [])

    # Prefer MANE Select or canonical transcript
    best_tc = None
    for tc in transcript_consequences:
        if tc.get("mane_select"):
            best_tc = tc
            break
        if tc.get("canonical") and not best_tc:
            best_tc = tc
    if not best_tc and transcript_consequences:
        best_tc = transcript_consequences[0]

    # Parse in-silico scores from the best transcript consequence
    scores = {}
    if best_tc:
        if "cadd_phred" in best_tc:
            scores["cadd_phred"] = best_tc["cadd_phred"]
            scores["cadd_raw"] = best_tc.get("cadd_raw")
        if "revel_score" in best_tc:
            scores["revel_score"] = best_tc["revel_score"]
        if "spliceai" in best_tc:
            scores["spliceai"] = best_tc["spliceai"]
        if "am_class" in best_tc:
            scores["alphamissense_class"] = best_tc["am_class"]
            scores["alphamissense_score"] = best_tc.get("am_pathogenicity")

    return {
        "found": True,
        "input": hgvs_notation,
        "variant_class": result.get("variant_class"),
        "most_severe_consequence": result.get("most_severe_consequence"),
        "assembly": result.get("assembly_name"),
        "location": f"{result.get('seq_region_name')}:{result.get('start')}-{result.get('end')}",
        "alleles": f"{result.get('allele_string')}",
        "transcript_consequence": {
            "transcript_id": best_tc.get("transcript_id") if best_tc else None,
            "gene_symbol": best_tc.get("gene_symbol") if best_tc else None,
            "consequence_terms": best_tc.get("consequence_terms", []) if best_tc else [],
            "impact": best_tc.get("impact") if best_tc else None,
            "amino_acids": best_tc.get("amino_acids") if best_tc else None,
            "codons": best_tc.get("codons") if best_tc else None,
            "protein_position": best_tc.get("protein_start") if best_tc else None,
            "exon": best_tc.get("exon") if best_tc else None,
            "hgvsc": best_tc.get("hgvsc") if best_tc else None,
            "hgvsp": best_tc.get("hgvsp") if best_tc else None,
            "domains": best_tc.get("domains", []) if best_tc else [],
            "mane_select": best_tc.get("mane_select") if best_tc else None,
            "canonical": best_tc.get("canonical") if best_tc else None,
        },
        "in_silico_scores": scores,
        "colocated_variants": [
            {
                "id": cv.get("id"),
                "allele_string": cv.get("allele_string"),
                "frequencies": cv.get("frequencies", {}),
                "clin_sig": cv.get("clin_sig"),
            }
            for cv in result.get("colocated_variants", [])
        ],
    }

```

Tool 3: search_pubmed

```python
"""Search PubMed for relevant genomic literature."""

import requests
import xml.etree.ElementTree as ET
from strands import tool

EUTILS_BASE = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils"


@tool
def search_pubmed(
    gene: str,
    variant: str = "",
    condition: str = "",
    max_results: int = 10,
) -> dict:
    """Search PubMed for published literature relevant to a genetic variant,
    gene, or clinical condition. Returns article titles, authors, abstracts,
    and PMIDs for citation in variant interpretation reports.

    Args:
        gene: Gene symbol (e.g., "COL3A1", "BRCA1").
        variant: Specific variant description to include in search
            (e.g., "p.Gly620Asp", "c.1859G>A"). Optional.
        condition: Associated clinical condition
            (e.g., "Ehlers-Danlos syndrome", "vascular EDS"). Optional.
        max_results: Maximum number of articles to return (default 10, max 50).

    Returns:
        Dictionary with list of articles including PMID, title, authors,
        journal, publication date, and abstract snippet.
    """
    # Build search query
    query_parts = [f"{gene}[Gene]"]
    if variant:
        query_parts.append(f'"{variant}"')
    if condition:
        query_parts.append(f'"{condition}"')

    query = " AND ".join(query_parts)

    # Step 1: Search for PMIDs
    search_url = f"{EUTILS_BASE}/esearch.fcgi"
    search_params = {
        "db": "pubmed",
        "term": query,
        "retmax": min(max_results, 50),
        "sort": "relevance",
        "retmode": "json",
    }

    try:
        search_resp = requests.get(search_url, params=search_params, timeout=15)
        search_resp.raise_for_status()
        search_data = search_resp.json()
    except Exception as e:
        return {"error": f"PubMed search failed: {str(e)}", "query": query}

    pmids = search_data.get("esearchresult", {}).get("idlist", [])
    total_count = int(
        search_data.get("esearchresult", {}).get("count", 0)
    )

    if not pmids:
        return {
            "found": False,
            "query": query,
            "total_results": 0,
            "message": "No PubMed articles found for this query.",
        }

    # Step 2: Fetch article details
    fetch_url = f"{EUTILS_BASE}/efetch.fcgi"
    fetch_params = {
        "db": "pubmed",
        "id": ",".join(pmids),
        "rettype": "xml",
        "retmode": "xml",
    }

    fetch_resp = requests.get(fetch_url, params=fetch_params, timeout=30)
    fetch_resp.raise_for_status()

    # Parse XML response
    root = ET.fromstring(fetch_resp.text)
    articles = []

    for article in root.findall(".//PubmedArticle"):
        medline = article.find(".//MedlineCitation")
        pmid = medline.findtext("PMID", "")
        art = medline.find("Article")

        title = art.findtext("ArticleTitle", "") if art is not None else ""
        journal = (
            art.find("Journal/Title").text
            if art is not None and art.find("Journal/Title") is not None
            else ""
        )

        # Extract authors
        author_list = art.findall(".//Author") if art is not None else []
        authors = []
        for auth in author_list[:5]:  # First 5 authors
            last = auth.findtext("LastName", "")
            init = auth.findtext("Initials", "")
            if last:
                authors.append(f"{last} {init}")
        if len(author_list) > 5:
            authors.append("et al.")

        # Extract year
        year = ""
        pub_date = art.find(".//PubDate") if art is not None else None
        if pub_date is not None:
            year = pub_date.findtext("Year", "")

        # Abstract
        abstract_parts = art.findall(".//AbstractText") if art is not None else []
        abstract = " ".join(
            (at.text or "") for at in abstract_parts
        )[:500]  # First 500 chars

        articles.append({
            "pmid": pmid,
            "title": title,
            "authors": ", ".join(authors),
            "journal": journal,
            "year": year,
            "abstract_snippet": abstract,
            "url": f"https://pubmed.ncbi.nlm.nih.gov/{pmid}/",
        })

    return {
        "found": True,
        "query": query,
        "total_results": total_count,
        "returned": len(articles),
        "articles": articles,
    }

```

Tool 4: apply_acmg_criteria

```python
"""Apply ACMG/AMP variant classification criteria."""

from strands import tool
from typing import Optional


@tool
def apply_acmg_criteria(
    gene: str,
    consequence: str,
    protein_change: str,
    clinvar_significance: str = "",
    gnomad_af: float = -1.0,
    cadd_phred: float = -1.0,
    revel_score: float = -1.0,
    spliceai_max: float = -1.0,
    alphamissense_score: float = -1.0,
    is_glycine_gxy: bool = False,
    is_triple_helical_domain: bool = False,
    protein_position: int = -1,
    total_protein_length: int = -1,
    known_pathogenic_at_position: bool = False,
    functional_study_result: str = "",
    domain_name: str = "",
    pubmed_articles_count: int = 0,
) -> dict:
    """Apply ACMG/AMP variant classification criteria to determine pathogenicity.

    Evaluates evidence across multiple ACMG criteria categories and combines them
    using the standard ACMG combining rules to produce a 5-tier classification:
    Pathogenic, Likely Pathogenic, VUS, Likely Benign, or Benign.

    Implements ClinGen-calibrated thresholds for PP3/BP4 (Pejaver et al., 2022).
    Includes EDS-specific modifications for collagen genes.

    Args:
        gene: Gene symbol (e.g., "COL3A1").
        consequence: VEP consequence type (e.g., "missense_variant", "frameshift_variant").
        protein_change: Protein change (e.g., "p.Gly620Asp").
        clinvar_significance: ClinVar clinical significance string (optional).
        gnomad_af: gnomAD overall allele frequency (-1 if unknown).
        cadd_phred: CADD Phred-scaled score (-1 if unknown).
        revel_score: REVEL score 0-1 (-1 if unknown).
        spliceai_max: Maximum SpliceAI delta score (-1 if unknown).
        alphamissense_score: AlphaMissense pathogenicity score 0-1 (-1 if unknown).
        is_glycine_gxy: Whether this is a glycine substitution in a collagen
            Gly-X-Y repeat (critical for EDS interpretation).
        is_triple_helical_domain: Whether the variant falls in the triple
            helical domain of a collagen gene.
        protein_position: Position in the protein sequence (-1 if unknown).
        total_protein_length: Total protein length (-1 if unknown).
        known_pathogenic_at_position: Whether a different pathogenic missense
            variant has been reported at this same amino acid position (for PS1).
        functional_study_result: Result of functional studies if available
            ("damaging", "benign", or empty string).
        domain_name: Name of the protein domain if variant overlaps one.
        pubmed_articles_count: Number of relevant PubMed articles found.

    Returns:
        Dictionary with ACMG classification, triggered criteria with evidence
        levels, point score, and detailed reasoning for each criterion.
    """
    # ACMG criteria evaluation — see Section 4 for full implementation
    criteria = {}
    reasoning = {}

    # --- Very Strong Pathogenic (PVS1) ---
    pvs1_genes = {"COL3A1", "COL5A1", "COL1A1", "COL1A2", "FBN1", "TGFBR1", "TGFBR2"}
    if consequence in ("frameshift_variant", "stop_gained", "splice_donor_variant",
                       "splice_acceptor_variant") and gene in pvs1_genes:
        criteria["PVS1"] = "very_strong"
        reasoning["PVS1"] = (
            f"Null variant ({consequence}) in {gene}, a gene where loss of function "
            f"is a known mechanism of disease."
        )

    # --- Strong Pathogenic (PS1) ---
    if known_pathogenic_at_position and consequence == "missense_variant":
        criteria["PS1"] = "strong"
        reasoning["PS1"] = (
            f"A different pathogenic missense variant has been reported at this "
            f"amino acid position ({protein_change})."
        )

    # --- Moderate Pathogenic (PM1, PM2, PM5) ---
    # PM1: Located in a mutational hotspot / well-established functional domain
    if is_triple_helical_domain and gene.startswith("COL"):
        criteria["PM1"] = "moderate"
        reasoning["PM1"] = (
            f"Variant located in the triple helical domain of {gene}. "
            f"This domain is a well-established hotspot for pathogenic variants "
            f"in collagen disorders."
        )
    elif domain_name:
        criteria["PM1"] = "moderate"
        reasoning["PM1"] = f"Variant located in functional domain: {domain_name}."

    # PM2: Absent or extremely low frequency in population databases
    if gnomad_af >= 0:
        if gnomad_af == 0:
            criteria["PM2"] = "moderate"  # Supporting per ClinGen recommendation
            reasoning["PM2"] = "Absent from gnomAD population database."
        elif gnomad_af < 0.00001:
            criteria["PM2"] = "supporting"  # Downgraded per ClinGen
            reasoning["PM2"] = f"Extremely rare in gnomAD (AF={gnomad_af:.6f})."

    # PM5: Novel missense at position where different pathogenic missense seen
    if (known_pathogenic_at_position and consequence == "missense_variant"
            and "PS1" not in criteria):
        criteria["PM5"] = "moderate"
        reasoning["PM5"] = (
            "Novel missense change at an amino acid residue where a different "
            "pathogenic missense change has been observed."
        )

    # --- Supporting Pathogenic (PP3) ---
    # ClinGen-calibrated thresholds (Pejaver et al., 2022, AJHG)
    pp3_evidence = []
    pp3_strength = "supporting"

    if revel_score >= 0:
        if revel_score >= 0.932:
            pp3_evidence.append(f"REVEL={revel_score:.3f} (≥0.932 = strong)")
            pp3_strength = "strong"  # ClinGen strong threshold
        elif revel_score >= 0.773:
            pp3_evidence.append(f"REVEL={revel_score:.3f} (≥0.773 = moderate)")
            if pp3_strength != "strong":
                pp3_strength = "moderate"
        elif revel_score >= 0.644:
            pp3_evidence.append(f"REVEL={revel_score:.3f} (≥0.644 = supporting)")

    if cadd_phred >= 0:
        if cadd_phred >= 25.3:
            pp3_evidence.append(f"CADD={cadd_phred:.1f} (≥25.3 = strong)")
            if pp3_strength != "strong":
                pp3_strength = "strong"
        elif cadd_phred >= 17.3:
            pp3_evidence.append(f"CADD={cadd_phred:.1f} (≥17.3 = supporting)")

    if alphamissense_score >= 0:
        if alphamissense_score >= 0.564:
            pp3_evidence.append(
                f"AlphaMissense={alphamissense_score:.3f} (≥0.564 = pathogenic)"
            )

    if pp3_evidence:
        criteria["PP3"] = pp3_strength
        reasoning["PP3"] = (
            f"Computational evidence supports pathogenicity: {'; '.join(pp3_evidence)}"
        )

    # --- Supporting Benign (BP4) ---
    bp4_evidence = []
    if revel_score >= 0 and revel_score <= 0.183:
        bp4_evidence.append(f"REVEL={revel_score:.3f} (≤0.183 = strong benign)")
    if cadd_phred >= 0 and cadd_phred <= 0.15:
        bp4_evidence.append(f"CADD={cadd_phred:.1f} (low)")

    if bp4_evidence and not pp3_evidence:
        criteria["BP4"] = "supporting"
        reasoning["BP4"] = (
            f"Computational evidence supports benign: {'; '.join(bp4_evidence)}"
        )

    # --- EDS-SPECIFIC: Glycine substitution in Gly-X-Y ---
    if is_glycine_gxy and gene in ("COL3A1", "COL5A1", "COL1A1", "COL1A2"):
        # Glycine substitutions in the triple helical domain of collagen genes
        # are functionally equivalent to null variants for vEDS (COL3A1)
        criteria["EDS_GlyXY"] = "very_strong"
        reasoning["EDS_GlyXY"] = (
            f"Glycine substitution in the Gly-X-Y repeat of {gene} triple helical "
            f"domain. This is a known pathogenic mechanism for collagen disorders — "
            f"glycine is the only amino acid small enough to fit in the interior of "
            f"the triple helix. This disrupts triple helix formation with "
            f"dominant-negative effect. Position {protein_position}/{total_protein_length} "
            f"({'C-terminal (more severe)' if protein_position > total_protein_length * 0.5 else 'N-terminal'}). "
            f"For COL3A1 vEDS: Gly substitutions in the triple helical domain are "
            f"classified as pathogenic per expert consensus."
        )

    # --- Functional studies (PS3/BS3) ---
    if functional_study_result == "damaging":
        criteria["PS3"] = "strong"
        reasoning["PS3"] = "Well-established functional studies show damaging effect."
    elif functional_study_result == "benign":
        criteria["BS3"] = "strong"
        reasoning["BS3"] = "Well-established functional studies show no damaging effect."

    # --- Classify using ACMG combining rules ---
    classification = _combine_acmg_criteria(criteria)

    return {
        "classification": classification,
        "criteria_triggered": criteria,
        "reasoning": reasoning,
        "criteria_count": {
            "pathogenic": sum(
                1 for k, v in criteria.items()
                if k.startswith(("PVS", "PS", "PM", "PP", "EDS"))
            ),
            "benign": sum(
                1 for k, v in criteria.items()
                if k.startswith(("BA", "BS", "BP"))
            ),
        },
        "note": (
            "This is a computational pre-classification. All variant "
            "classifications must be reviewed by a qualified clinical "
            "geneticist before clinical use."
        ),
    }


def _combine_acmg_criteria(criteria: dict) -> str:
    """Combine ACMG criteria using standard rules to determine classification.

    Rules per Richards et al., 2015:

    Pathogenic:
      (i)   1 Very Strong + ≥1 Strong
      (ii)  1 Very Strong + ≥2 Moderate
      (iii) 1 Very Strong + 1 Moderate + 1 Supporting
      (iv)  1 Very Strong + ≥2 Supporting
      (v)   ≥2 Strong
      (vi)  1 Strong + ≥3 Moderate
      (vii) 1 Strong + 2 Moderate + ≥2 Supporting
      (viii) 1 Strong + 1 Moderate + ≥4 Supporting

    Likely Pathogenic:
      (i)   1 Very Strong + 1 Moderate
      (ii)  1 Strong + 1-2 Moderate
      (iii) 1 Strong + ≥2 Supporting
      (iv)  ≥3 Moderate
      (v)   2 Moderate + ≥2 Supporting
      (vi)  1 Moderate + ≥4 Supporting
    """
    very_strong = sum(1 for v in criteria.values() if v == "very_strong")
    strong = sum(
        1 for k, v in criteria.items()
        if v == "strong" and not k.startswith(("BA", "BS", "BP"))
    )
    moderate = sum(
        1 for k, v in criteria.items()
        if v == "moderate" and not k.startswith(("BA", "BS", "BP"))
    )
    supporting = sum(
        1 for k, v in criteria.items()
        if v == "supporting" and not k.startswith(("BA", "BS", "BP"))
    )

    # Benign criteria
    ba = sum(1 for k, v in criteria.items() if k.startswith("BA"))
    bs = sum(1 for k, v in criteria.items() if k.startswith("BS"))
    bp = sum(1 for k, v in criteria.items() if k.startswith("BP"))

    # Benign classification
    if ba >= 1:
        return "Benign"
    if bs >= 2:
        return "Benign"
    if bs >= 1 and bp >= 1:
        return "Likely Benign"
    if bp >= 2:
        return "Likely Benign"

    # Pathogenic classification
    if very_strong >= 1:
        if strong >= 1:
            return "Pathogenic"
        if moderate >= 2:
            return "Pathogenic"
        if moderate >= 1 and supporting >= 1:
            return "Pathogenic"
        if supporting >= 2:
            return "Pathogenic"
        if moderate >= 1:
            return "Likely Pathogenic"

    if strong >= 2:
        return "Pathogenic"
    if strong >= 1:
        if moderate >= 3:
            return "Pathogenic"
        if moderate >= 2 and supporting >= 2:
            return "Pathogenic"
        if moderate >= 1 and supporting >= 4:
            return "Pathogenic"
        if moderate >= 1:
            return "Likely Pathogenic"
        if supporting >= 2:
            return "Likely Pathogenic"

    if moderate >= 3:
        return "Likely Pathogenic"
    if moderate >= 2 and supporting >= 2:
        return "Likely Pathogenic"
    if moderate >= 1 and supporting >= 4:
        return "Likely Pathogenic"

    return "VUS (Variant of Uncertain Significance)"

```

### Complete Agent Assembly

```python
"""Genomic Variant Interpretation Agent — main entry point."""

from strands import Agent
from strands.models.bedrock import BedrockModel

# Import tools
from tools.annotation_store import query_annotation_store
from tools.vep import query_vep
from tools.pubmed import search_pubmed
from tools.acmg import apply_acmg_criteria

# Configure the model
model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-5-20250514-v1:0",
    region_name="us-east-1",
    max_tokens=8192,
)

# Create the agent
variant_agent = Agent(
    model=model,
    system_prompt=VARIANT_INTERPRETER_SYSTEM_PROMPT,  # From Section 3 above
    tools=[
        query_annotation_store,
        query_vep,
        search_pubmed,
        apply_acmg_criteria,
    ],
)

# Invoke the agent
response = variant_agent(
    "Interpret the variant COL3A1 c.1859G>A (p.Gly620Asp) in the context of "
    "vascular Ehlers-Danlos syndrome. Provide full ACMG classification."
)

print(response)

```

### Tool Schemas (JSON — for AgentCore Harness Inline Functions)

When deploying on AgentCore Runtime (as opposed to running Strands locally), tools are defined as inline functions with JSON schemas. Here are the complete schemas:

```json
{
  "tools": [
    {
      "type": "inline_function",
      "name": "query_annotation_store",
      "config": {
        "inlineFunction": {
          "description": "Query ClinVar and gnomAD annotations for a genomic variant from the HealthOmics annotation store via Athena. Returns clinical significance, allele frequencies, and variant metadata.",
          "inputSchema": {
            "type": "object",
            "properties": {
              "chrom": {
                "type": "string",
                "description": "Chromosome (e.g., '2', 'X'). Do not include 'chr' prefix."
              },
              "pos": {
                "type": "integer",
                "description": "Genomic position in GRCh38 coordinates."
              },
              "ref": {
                "type": "string",
                "description": "Reference allele."
              },
              "alt": {
                "type": "string",
                "description": "Alternate allele."
              },
              "gene": {
                "type": "string",
                "description": "Gene symbol for broader gene-level queries (optional)."
              }
            },
            "required": ["chrom", "pos", "ref", "alt"]
          }
        }
      }
    },
    {
      "type": "inline_function",
      "name": "query_vep",
      "config": {
        "inlineFunction": {
          "description": "Query the Ensembl Variant Effect Predictor (VEP) REST API for comprehensive variant annotation. Returns consequence type, protein impact, CADD/REVEL/SpliceAI/AlphaMissense scores, overlapping domains, and co-located known variants.",
          "inputSchema": {
            "type": "object",
            "properties": {
              "hgvs_notation": {
                "type": "string",
                "description": "Variant in HGVS notation. Supports coding (c.), genomic (g.), or protein (p.) with transcript/gene reference. Examples: 'ENST00000304636:c.1859G>A', 'COL3A1:c.1859G>A'"
              },
              "include_cadd": {
                "type": "boolean",
                "description": "Include CADD deleteriousness scores.",
                "default": true
              },
              "include_revel": {
                "type": "boolean",
                "description": "Include REVEL pathogenicity scores.",
                "default": true
              },
              "include_spliceai": {
                "type": "boolean",
                "description": "Include SpliceAI splice prediction scores.",
                "default": true
              },
              "include_alphamissense": {
                "type": "boolean",
                "description": "Include AlphaMissense pathogenicity scores.",
                "default": true
              }
            },
            "required": ["hgvs_notation"]
          }
        }
      }
    },
    {
      "type": "inline_function",
      "name": "search_pubmed",
      "config": {
        "inlineFunction": {
          "description": "Search PubMed for published literature relevant to a genetic variant, gene, or clinical condition. Returns article titles, authors, abstracts, and PMIDs for citation.",
          "inputSchema": {
            "type": "object",
            "properties": {
              "gene": {
                "type": "string",
                "description": "Gene symbol (e.g., 'COL3A1', 'BRCA1')."
              },
              "variant": {
                "type": "string",
                "description": "Specific variant description (e.g., 'p.Gly620Asp'). Optional."
              },
              "condition": {
                "type": "string",
                "description": "Associated clinical condition (e.g., 'vascular Ehlers-Danlos syndrome'). Optional."
              },
              "max_results": {
                "type": "integer",
                "description": "Maximum number of articles to return (default 10).",
                "default": 10
              }
            },
            "required": ["gene"]
          }
        }
      }
    },
    {
      "type": "inline_function",
      "name": "apply_acmg_criteria",
      "config": {
        "inlineFunction": {
          "description": "Apply ACMG/AMP variant classification criteria using collected evidence. Returns 5-tier classification (Pathogenic/Likely Pathogenic/VUS/Likely Benign/Benign) with triggered criteria and reasoning. Uses ClinGen-calibrated thresholds and includes EDS-specific collagen rules.",
          "inputSchema": {
            "type": "object",
            "properties": {
              "gene": {"type": "string", "description": "Gene symbol."},
              "consequence": {"type": "string", "description": "VEP consequence type (e.g., 'missense_variant')."},
              "protein_change": {"type": "string", "description": "Protein change (e.g., 'p.Gly620Asp')."},
              "clinvar_significance": {"type": "string", "description": "ClinVar clinical significance."},
              "gnomad_af": {"type": "number", "description": "gnomAD allele frequency (-1 if unknown)."},
              "cadd_phred": {"type": "number", "description": "CADD Phred-scaled score (-1 if unknown)."},
              "revel_score": {"type": "number", "description": "REVEL score 0-1 (-1 if unknown)."},
              "spliceai_max": {"type": "number", "description": "Max SpliceAI delta score (-1 if unknown)."},
              "alphamissense_score": {"type": "number", "description": "AlphaMissense score 0-1 (-1 if unknown)."},
              "is_glycine_gxy": {"type": "boolean", "description": "Whether this is a Gly substitution in a collagen Gly-X-Y repeat."},
              "is_triple_helical_domain": {"type": "boolean", "description": "Whether variant is in triple helical domain."},
              "protein_position": {"type": "integer", "description": "Position in protein sequence (-1 if unknown)."},
              "total_protein_length": {"type": "integer", "description": "Total protein length (-1 if unknown)."},
              "known_pathogenic_at_position": {"type": "boolean", "description": "Whether a different pathogenic missense is known at this position."},
              "functional_study_result": {"type": "string", "description": "Functional study result: 'damaging', 'benign', or empty."},
              "domain_name": {"type": "string", "description": "Name of overlapping protein domain."},
              "pubmed_articles_count": {"type": "integer", "description": "Number of relevant PubMed articles found."}
            },
            "required": ["gene", "consequence", "protein_change"]
          }
        }
      }
    }
  ]
}

```

---

## 4. ACMG Rules Engine

### Background: ACMG/AMP Framework

The American College of Medical Genetics and Genomics (ACMG) and the Association for Molecular Pathology (AMP) published joint guidelines (Richards et al., 2015, *Genetics in Medicine* 17:405–424) establishing a standardized 5-tier classification system:

| Classification | Abbreviation | Actionability |
| --- | --- | --- |
| **Pathogenic** | Class 5 | Return to patient, clinical action |
| **Likely Pathogenic** | Class 4 | Return to patient, clinical action with caveats |
| **VUS** | Class 3 | Report, but insufficient evidence for action |
| **Likely Benign** | Class 2 | Typically not reported |
| **Benign** | Class 1 | Not reported |

Evidence criteria are categorized by **type** (pathogenic or benign) and **strength** (stand-alone, very strong, strong, moderate, supporting).

### ClinGen-Calibrated Thresholds for PP3/BP4

The ClinGen Sequence Variant Interpretation (SVI) Working Group published calibrated thresholds for computational predictors (Pejaver et al., 2022, *AJHG* 111:2504–2515). These allow PP3 and BP4 to be applied at strengths **above** "supporting" when prediction scores reach validated thresholds:

```
┌─────────────────────────────────────────────────────────────────────┐
│            ClinGen-Calibrated PP3/BP4 Thresholds                   │
├──────────────┬──────────┬──────────┬───────────┬──────────────────┤
│ Predictor    │ PP3      │ PP3      │ PP3       │ BP4              │
│              │Supporting│ Moderate │ Strong    │ Supporting/Strong│
├──────────────┼──────────┼──────────┼───────────┼──────────────────┤
│ REVEL        │ ≥ 0.644  │ ≥ 0.773  │ ≥ 0.932   │ ≤ 0.290 / ≤0.183│
│ CADD (Phred) │ ≥ 17.3   │ ≥ 21.1   │ ≥ 25.3    │ ≤ 7.7 / ≤ 0.15  │
│ BayesDel     │ ≥ 0.13   │ ≥ 0.27   │ ≥ 0.50    │ ≤ -0.18 / ≤-0.36│
│ AlphaMissense│ ≥ 0.340  │ ≥ 0.564  │ ≥ 0.946   │ ≤ 0.113         │
│ MutPred2     │ ≥ 0.61   │ ≥ 0.79   │ ≥ 0.93    │ ≤ 0.01          │
│ VEST4        │ ≥ 0.764  │ ≥ 0.861  │ ≥ 0.966   │ ≤ 0.449 / ≤0.29 │
├──────────────┴──────────┴──────────┴───────────┴──────────────────┤
│ SpliceAI     │ ≥ 0.2 (supporting) │ ≥ 0.5 (strong)               │
│ (for splice  │ ≥ 0.1 (for BP4)    │ ≥ 0.8 (very strong for PVS1) │
│  variants)   │                     │                               │
└─────────────────────────────────────────────────────────────────────┘

```

### Python Implementation of Key ACMG Criteria

```python
"""
ACMG Rules Engine — detailed implementation of key criteria.
Reference: Richards et al., 2015; Pejaver et al., 2022; ClinGen SVI recommendations.
"""

from dataclasses import dataclass, field
from enum import Enum
from typing import Optional


class EvidenceStrength(Enum):
    STAND_ALONE = "stand_alone"
    VERY_STRONG = "very_strong"
    STRONG = "strong"
    MODERATE = "moderate"
    SUPPORTING = "supporting"


@dataclass
class ACMGEvidence:
    criterion: str
    strength: EvidenceStrength
    met: bool
    reasoning: str
    source: str = ""


@dataclass
class VariantEvidence:
    """All evidence collected for a variant."""
    gene: str
    consequence: str
    protein_change: str
    transcript: str = ""
    # Population
    gnomad_af: float = -1.0
    gnomad_af_popmax: float = -1.0
    # Computational
    cadd_phred: float = -1.0
    revel_score: float = -1.0
    spliceai_max: float = -1.0
    alphamissense_score: float = -1.0
    # Structural
    is_glycine_gxy: bool = False
    is_triple_helical_domain: bool = False
    protein_position: int = -1
    total_protein_length: int = -1
    domain_name: str = ""
    # Clinical
    clinvar_significance: str = ""
    clinvar_review_status: str = ""
    known_pathogenic_same_aa: bool = False
    known_pathogenic_same_change: bool = False
    # Functional
    functional_study_result: str = ""
    # Segregation / De novo
    segregation_data: str = ""
    de_novo_confirmed: bool = False


# ─── Collagen-specific gene lists ───────────────────────────────────
LOF_GENES = {
    "COL3A1", "COL5A1", "COL5A2", "COL1A1", "COL1A2",
    "FBN1", "TGFBR1", "TGFBR2", "SMAD3", "TNXB",
    "PLOD1", "ADAMTS2", "FKBP14", "COL12A1",
}

# COL3A1 triple helical domain: residues 168-1196 (Gly-X-Y repeats)
COL3A1_TRIPLE_HELIX = (168, 1196)
COL3A1_TOTAL_LENGTH = 1466


def evaluate_pvs1(ev: VariantEvidence) -> ACMGEvidence:
    """PVS1: Null variant in a gene where LOF is a known disease mechanism.

    Null variants: nonsense, frameshift, canonical ±1 or ±2 splice sites,
    initiation codon, single/multi-exon deletion.
    """
    null_consequences = {
        "frameshift_variant", "stop_gained", "splice_donor_variant",
        "splice_acceptor_variant", "start_lost", "transcript_ablation",
    }

    is_null = ev.consequence in null_consequences
    is_lof_gene = ev.gene in LOF_GENES

    # Special case: COL3A1 null → haploinsufficiency → milder vEDS
    # Still PVS1, but note clinical distinction
    note = ""
    if ev.gene == "COL3A1" and is_null:
        note = (
            " Note: COL3A1 haploinsufficiency variants cause a milder vEDS "
            "phenotype compared to dominant-negative glycine substitutions. "
            "Still pathogenic, but clinical management may differ."
        )

    met = is_null and is_lof_gene
    return ACMGEvidence(
        criterion="PVS1",
        strength=EvidenceStrength.VERY_STRONG,
        met=met,
        reasoning=(
            f"{'Null' if is_null else 'Non-null'} variant ({ev.consequence}) "
            f"in {ev.gene} ({'known' if is_lof_gene else 'not established'} "
            f"LOF gene).{note}"
        ),
        source="Richards et al., 2015",
    )


def evaluate_ps1(ev: VariantEvidence) -> ACMGEvidence:
    """PS1: Same amino acid change as a previously established pathogenic variant.

    The amino acid change must be the same, not just at the same position
    (that's PM5). Must verify the nucleotide change does not affect splicing.
    """
    met = ev.known_pathogenic_same_change and ev.consequence == "missense_variant"
    return ACMGEvidence(
        criterion="PS1",
        strength=EvidenceStrength.STRONG,
        met=met,
        reasoning=(
            f"{'Same' if met else 'No same'} amino acid change as an "
            f"established pathogenic variant at {ev.protein_change}."
        ),
        source="Richards et al., 2015",
    )


def evaluate_pm1(ev: VariantEvidence) -> ACMGEvidence:
    """PM1: Located in a mutational hot spot and/or critical functional domain.

    EDS-specific: The triple helical domain (Gly-X-Y repeats) in collagen
    genes is a well-established hotspot for pathogenic variants.
    """
    in_hotspot = False
    detail = ""

    if ev.gene == "COL3A1" and ev.protein_position >= 0:
        start, end = COL3A1_TRIPLE_HELIX
        if start <= ev.protein_position <= end:
            in_hotspot = True
            # Position within triple helix affects severity
            relative_pos = (ev.protein_position - start) / (end - start)
            severity = "C-terminal (more severe)" if relative_pos > 0.5 else "N-terminal"
            detail = (
                f"Position {ev.protein_position} in COL3A1 triple helical domain "
                f"(residues {start}-{end}). {severity} region. "
                f"This domain contains the obligatory Gly-X-Y repeats essential "
                f"for triple helix formation."
            )
    elif ev.is_triple_helical_domain:
        in_hotspot = True
        detail = f"Located in triple helical domain of {ev.gene}."
    elif ev.domain_name:
        in_hotspot = True
        detail = f"Located in functional domain: {ev.domain_name}."

    return ACMGEvidence(
        criterion="PM1",
        strength=EvidenceStrength.MODERATE,
        met=in_hotspot,
        reasoning=detail or "No established hotspot/domain overlap identified.",
        source="Richards et al., 2015; Byers et al., 2017",
    )


def evaluate_pm2(ev: VariantEvidence) -> ACMGEvidence:
    """PM2: Absent from controls (or at extremely low frequency).

    ClinGen SVI recommendation: PM2 should be applied at supporting level
    when absent from gnomAD, not at moderate level, due to database limitations.

    For dominant conditions with high penetrance (like vEDS):
    - Absent from gnomAD → PM2_supporting
    - AF < 0.00001 → PM2_supporting
    """
    met = False
    strength = EvidenceStrength.SUPPORTING  # ClinGen downgrade
    detail = ""

    if ev.gnomad_af < 0:
        detail = "gnomAD allele frequency not available."
    elif ev.gnomad_af == 0:
        met = True
        detail = "Absent from gnomAD (all populations). Applied at supporting level per ClinGen SVI recommendation."
    elif ev.gnomad_af < 0.00001:
        met = True
        detail = f"Extremely rare in gnomAD (AF={ev.gnomad_af:.2e}). Applied at supporting level."
    else:
        detail = f"Present in gnomAD at AF={ev.gnomad_af:.2e}. PM2 not met."

    return ACMGEvidence(
        criterion="PM2",
        strength=strength,
        met=met,
        reasoning=detail,
        source="Richards et al., 2015; ClinGen SVI, 2020",
    )


def evaluate_pp3(ev: VariantEvidence) -> ACMGEvidence:
    """PP3: Computational evidence supports a deleterious effect.

    Uses ClinGen-calibrated thresholds (Pejaver et al., 2022, AJHG).
    Multiple predictors can be assessed; use the strongest evidence level.
    """
    evidence_lines = []
    best_strength = EvidenceStrength.SUPPORTING

    # REVEL — recommended as primary predictor by ClinGen
    if ev.revel_score >= 0:
        if ev.revel_score >= 0.932:
            evidence_lines.append(
                f"REVEL={ev.revel_score:.3f} (≥0.932 → PP3_strong)"
            )
            best_strength = EvidenceStrength.STRONG
        elif ev.revel_score >= 0.773:
            evidence_lines.append(
                f"REVEL={ev.revel_score:.3f} (≥0.773 → PP3_moderate)"
            )
            if best_strength != EvidenceStrength.STRONG:
                best_strength = EvidenceStrength.MODERATE
        elif ev.revel_score >= 0.644:
            evidence_lines.append(
                f"REVEL={ev.revel_score:.3f} (≥0.644 → PP3_supporting)"
            )

    # CADD
    if ev.cadd_phred >= 0:
        if ev.cadd_phred >= 25.3:
            evidence_lines.append(
                f"CADD Phred={ev.cadd_phred:.1f} (≥25.3 → PP3_strong)"
            )
            best_strength = EvidenceStrength.STRONG
        elif ev.cadd_phred >= 17.3:
            evidence_lines.append(
                f"CADD Phred={ev.cadd_phred:.1f} (≥17.3 → PP3_supporting)"
            )

    # SpliceAI (for splice-proximal variants)
    if ev.spliceai_max >= 0 and ev.consequence in (
        "splice_region_variant", "splice_donor_variant",
        "splice_acceptor_variant", "synonymous_variant",
        "intron_variant",
    ):
        if ev.spliceai_max >= 0.5:
            evidence_lines.append(
                f"SpliceAI max Δ={ev.spliceai_max:.2f} (≥0.5 → PP3_strong)"
            )
            best_strength = EvidenceStrength.STRONG
        elif ev.spliceai_max >= 0.2:
            evidence_lines.append(
                f"SpliceAI max Δ={ev.spliceai_max:.2f} (≥0.2 → PP3_supporting)"
            )

    # AlphaMissense
    if ev.alphamissense_score >= 0:
        if ev.alphamissense_score >= 0.564:
            evidence_lines.append(
                f"AlphaMissense={ev.alphamissense_score:.3f} (≥0.564 → likely pathogenic)"
            )

    met = len(evidence_lines) > 0
    return ACMGEvidence(
        criterion="PP3",
        strength=best_strength,
        met=met,
        reasoning=(
            "Computational predictions: " + "; ".join(evidence_lines)
            if evidence_lines
            else "No computational predictions available or all below threshold."
        ),
        source="Pejaver et al., 2022; ClinGen SVI",
    )


def evaluate_bp4(ev: VariantEvidence) -> ACMGEvidence:
    """BP4: Computational evidence suggests no impact on gene/gene product.

    Applied when multiple predictors concordantly predict benign effect.
    """
    evidence_lines = []

    if ev.revel_score >= 0 and ev.revel_score <= 0.290:
        evidence_lines.append(f"REVEL={ev.revel_score:.3f} (≤0.290 → benign)")
    if ev.cadd_phred >= 0 and ev.cadd_phred <= 7.7:
        evidence_lines.append(f"CADD Phred={ev.cadd_phred:.1f} (≤7.7 → benign)")

    met = len(evidence_lines) >= 1
    return ACMGEvidence(
        criterion="BP4",
        strength=EvidenceStrength.SUPPORTING,
        met=met,
        reasoning=(
            "Computational predictions support benign: " + "; ".join(evidence_lines)
            if evidence_lines
            else "No computational evidence for benign classification."
        ),
        source="Pejaver et al., 2022; ClinGen SVI",
    )


def evaluate_eds_gly_xy(ev: VariantEvidence) -> Optional[ACMGEvidence]:
    """EDS-specific: Glycine substitution in Gly-X-Y repeat.

    For COL3A1: Glycine substitutions in the triple helical domain are the
    most common cause of vascular EDS. The triple helix requires glycine
    (the smallest amino acid) at every third position. Any substitution
    disrupts triple helix formation with a dominant-negative effect.

    This is effectively equivalent to PVS1 for vEDS classification.
    Many ClinGen expert panels classify these as pathogenic on this
    criterion alone when combined with PM2.

    Key paper: Pepin et al., 2014, Genet Med 16:881-888
    "Molecular diagnosis in Vascular Ehlers-Danlos Syndrome predicts
    pattern of arterial involvement and outcomes"
    """
    if not ev.is_glycine_gxy:
        return None
    if ev.gene not in ("COL3A1", "COL5A1", "COL1A1", "COL1A2"):
        return None

    severity_note = ""
    if ev.gene == "COL3A1" and ev.protein_position > 0:
        # C-terminal glycine substitutions in COL3A1 are associated with
        # more severe phenotype and earlier onset
        if ev.protein_position > (COL3A1_TRIPLE_HELIX[0] + COL3A1_TRIPLE_HELIX[1]) / 2:
            severity_note = (
                " C-terminal location suggests potentially more severe phenotype "
                "with earlier onset of vascular events (Pepin et al., 2014)."
            )
        else:
            severity_note = " N-terminal location may be associated with later onset."

    # Determine which amino acid glycine is replaced with
    replacement = ""
    if ev.protein_change:
        # Parse p.Gly620Asp → Asp
        parts = ev.protein_change.replace("p.", "")
        if len(parts) > 6:
            replacement = parts[6:]  # After "GlyNNN"

    return ACMGEvidence(
        criterion="EDS_GlyXY",
        strength=EvidenceStrength.VERY_STRONG,
        met=True,
        reasoning=(
            f"Glycine substitution (Gly→{replacement}) at position "
            f"{ev.protein_position} in the Gly-X-Y repeat of {ev.gene} "
            f"triple helical domain. Glycine is the only amino acid small "
            f"enough to occupy the sterically restricted interior of the "
            f"collagen triple helix. This substitution disrupts triple helix "
            f"folding with a dominant-negative mechanism — the mutant chain "
            f"is incorporated but prevents normal helix formation, which is "
            f"more deleterious than haploinsufficiency.{severity_note}"
        ),
        source="Pepin et al., 2014; Byers et al., 2017; Malfait et al., 2017",
    )

```

### Mapping In-Silico Scores to ACMG Evidence Levels

```
┌───────────────────────────────────────────────────────────────────────────┐
│                 In-Silico Score → ACMG Evidence Mapping                  │
│                                                                           │
│   REVEL Score                          ACMG Evidence Level               │
│   ═══════════                          ══════════════════                 │
│   0.0 ├────────────────────┤ 0.290    BP4 (supporting benign)            │
│       │  ████████████████  │                                             │
│   0.290 ├──────────────────┤ 0.644   No evidence (gray zone)             │
│         │                  │                                             │
│   0.644 ├──────────────────┤ 0.773   PP3_supporting                      │
│         │  ▓▓▓▓▓▓▓▓▓▓▓▓▓  │                                             │
│   0.773 ├──────────────────┤ 0.932   PP3_moderate                        │
│         │  ████████████████│                                             │
│   0.932 ├──────────────────┤ 1.0     PP3_strong                          │
│         │  ████████████████│                                             │
│                                                                           │
│   CADD Phred                           ACMG Evidence Level               │
│   ══════════                           ══════════════════                 │
│   0.0 ├────┤ 0.15                     BP4 (strong benign)                │
│   0.15 ├───┤ 7.7                      BP4 (supporting benign)            │
│   7.7 ├────────┤ 17.3                 No evidence (gray zone)            │
│   17.3 ├───────┤ 25.3                 PP3_supporting                     │
│   25.3 ├───────────────┤ 40+          PP3_strong                         │
│                                                                           │
│   SpliceAI max Δ                       ACMG Evidence Level               │
│   ══════════════                       ══════════════════                 │
│   0.0 ├──┤ 0.1                        BP4 (supporting benign)            │
│   0.1 ├──┤ 0.2                        Indeterminate                      │
│   0.2 ├──────┤ 0.5                    PP3_supporting / PVS1_supporting   │
│   0.5 ├──────┤ 0.8                    PP3_strong / PVS1_moderate         │
│   0.8 ├──────────────┤ 1.0            PVS1_strong                        │
└───────────────────────────────────────────────────────────────────────────┘

```

### EDS-Specific ACMG Modifications

| Criterion | Standard ACMG | EDS / Collagen Modification |
| --- | --- | --- |
| **PVS1** | Null variant in LOF gene | Applied for COL3A1 (but note: haploinsufficiency variants → milder vEDS vs. dominant-negative Gly substitutions → classical severe vEDS) |
| **PM1** | Hotspot / functional domain | **Triple helical domain** of collagen genes (Gly-X-Y repeats). For COL3A1: residues 168–1196. Position within the helix matters — C-terminal substitutions correlate with earlier/more severe vascular events |
| **PM5** | Novel missense at known pathogenic position | Highly relevant for collagen Gly-X-Y positions — many different substitutions at the same Gly position are pathogenic |
| **PP3** | Computational evidence | Standard thresholds apply, but collagen Gly→X substitutions typically score very high on all predictors |
| **EDS_GlyXY** | *Not standard ACMG* | **Custom criterion:** Glycine substitution in Gly-X-Y repeat of COL3A1/COL5A1/COL1A1/COL1A2. Applied at **very strong** evidence level, functionally equivalent to PVS1 for classification purposes |

---

## 5. Example Walkthrough: COL3A1 c.1859G>A

### The Variant

- **Gene:** COL3A1 (Collagen Type III Alpha 1 Chain)
- **Transcript:** NM_000090.4 (MANE Select)
- **cDNA change:** c.1859G>A
- **Protein change:** p.Gly620Asp (Glycine → Aspartic Acid at position 620)
- **Genomic (GRCh38):** chr2:188,974,589 C>T (reverse strand)
- **Associated condition:** Vascular Ehlers-Danlos Syndrome (vEDS, OMIM #130050)

### Agent Tool Call Sequence

```
User: "Interpret COL3A1 c.1859G>A (p.Gly620Asp) for vascular EDS"

╔══════════════════════════════════════════════════════════════════╗
║  Agent Reasoning Loop — Step 1: Gather Evidence                 ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  → Tool Call 1: query_annotation_store                           ║
║    {                                                             ║
║      "chrom": "2",                                               ║
║      "pos": 188974589,                                           ║
║      "ref": "C",                                                 ║
║      "alt": "T",                                                 ║
║      "gene": "COL3A1"                                            ║
║    }                                                             ║
║                                                                  ║
║  → Tool Call 2: query_vep (parallel)                             ║
║    {                                                             ║
║      "hgvs_notation": "NM_000090.4:c.1859G>A",                  ║
║      "include_cadd": true,                                       ║
║      "include_revel": true,                                      ║
║      "include_spliceai": true,                                   ║
║      "include_alphamissense": true                               ║
║    }                                                             ║
║                                                                  ║
║  → Tool Call 3: search_pubmed (parallel)                         ║
║    {                                                             ║
║      "gene": "COL3A1",                                           ║
║      "variant": "p.Gly620Asp",                                   ║
║      "condition": "vascular Ehlers-Danlos syndrome",              ║
║      "max_results": 10                                           ║
║    }                                                             ║
║                                                                  ║
╠══════════════════════════════════════════════════════════════════╣
║  Tool Results                                                    ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  query_annotation_store result:                                  ║
║  {                                                               ║
║    "found": true,                                                ║
║    "annotations": [{                                             ║
║      "clinvar_clnsig": "Pathogenic/Likely_pathogenic",           ║
║      "clinvar_clnrevstat": "criteria_provided,_multiple_         ║
║                             submitters,_no_conflicts",           ║
║      "clinvar_clndn": "Ehlers-Danlos_syndrome_vascular_type",   ║
║      "gnomad_af": "0.0",                                        ║
║      "cadd_phred": "33.0",                                      ║
║      "revel_score": "0.968"                                      ║
║    }]                                                            ║
║  }                                                               ║
║                                                                  ║
║  query_vep result:                                               ║
║  {                                                               ║
║    "found": true,                                                ║
║    "most_severe_consequence": "missense_variant",                ║
║    "transcript_consequence": {                                   ║
║      "gene_symbol": "COL3A1",                                   ║
║      "consequence_terms": ["missense_variant"],                  ║
║      "impact": "MODERATE",                                       ║
║      "amino_acids": "G/D",                                      ║
║      "protein_position": 620,                                    ║
║      "exon": "26/52",                                            ║
║      "hgvsc": "NM_000090.4:c.1859G>A",                          ║
║      "hgvsp": "NP_000081.1:p.Gly620Asp",                        ║
║      "domains": [                                                ║
║        {"db": "Pfam", "name": "Collagen triple helix repeat"}   ║
║      ]                                                           ║
║    },                                                            ║
║    "in_silico_scores": {                                         ║
║      "cadd_phred": 33.0,                                        ║
║      "revel_score": 0.968,                                       ║
║      "alphamissense_class": "likely_pathogenic",                 ║
║      "alphamissense_score": 0.987                                ║
║    },                                                            ║
║    "colocated_variants": [{                                      ║
║      "id": "rs387906622",                                        ║
║      "clin_sig": ["pathogenic"]                                  ║
║    }]                                                            ║
║  }                                                               ║
║                                                                  ║
║  search_pubmed result:                                           ║
║  {                                                               ║
║    "found": true,                                                ║
║    "total_results": 47,                                          ║
║    "articles": [                                                 ║
║      {                                                           ║
║        "pmid": "24599722",                                       ║
║        "title": "Molecular diagnosis in Vascular Ehlers-Danlos   ║
║                  Syndrome Predicts Pattern of Arterial            ║
║                  Involvement and Outcomes",                      ║
║        "authors": "Pepin MG, Schwarze U, et al.",                ║
║        "journal": "Genet Med",                                   ║
║        "year": "2014"                                            ║
║      },                                                          ║
║      {                                                           ║
║        "pmid": "34773761",                                       ║
║        "title": "Four decades in the making: Collagen III and    ║
║                  mechanisms of vascular Ehlers Danlos Syndrome",  ║
║        "authors": "Bowen CJ, Giadrossi JJ, et al.",             ║
║        "journal": "Matrix Biol Plus",                            ║
║        "year": "2021"                                            ║
║      },                                                          ║
║      ...                                                         ║
║    ]                                                             ║
║  }                                                               ║
║                                                                  ║
╠══════════════════════════════════════════════════════════════════╣
║  Agent Reasoning Loop — Step 2: Apply ACMG Classification       ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  → Tool Call 4: apply_acmg_criteria                              ║
║    {                                                             ║
║      "gene": "COL3A1",                                           ║
║      "consequence": "missense_variant",                          ║
║      "protein_change": "p.Gly620Asp",                            ║
║      "clinvar_significance": "Pathogenic/Likely_pathogenic",     ║
║      "gnomad_af": 0.0,                                          ║
║      "cadd_phred": 33.0,                                        ║
║      "revel_score": 0.968,                                       ║
║      "spliceai_max": -1.0,                                      ║
║      "alphamissense_score": 0.987,                               ║
║      "is_glycine_gxy": true,                                    ║
║      "is_triple_helical_domain": true,                           ║
║      "protein_position": 620,                                    ║
║      "total_protein_length": 1466,                               ║
║      "known_pathogenic_at_position": true,                       ║
║      "domain_name": "Collagen triple helix repeat",              ║
║      "pubmed_articles_count": 47                                 ║
║    }                                                             ║
║                                                                  ║
║  apply_acmg_criteria result:                                     ║
║  {                                                               ║
║    "classification": "Pathogenic",                               ║
║    "criteria_triggered": {                                       ║
║      "EDS_GlyXY": "very_strong",                                ║
║      "PM1": "moderate",                                          ║
║      "PM2": "supporting",                                        ║
║      "PP3": "strong",                                            ║
║      "PS1": "strong"                                             ║
║    }                                                             ║
║  }                                                               ║
║                                                                  ║
╠══════════════════════════════════════════════════════════════════╣
║  Agent Reasoning Loop — Step 3: Synthesize Report               ║
╚══════════════════════════════════════════════════════════════════╝

```

### Expected Agent Output

```markdown
# Variant Interpretation Report

## Variant Identification
| Field | Value |
|-------|-------|
| **Gene** | COL3A1 (Collagen Type III Alpha 1 Chain) |
| **Transcript** | NM_000090.4 (MANE Select) |
| **cDNA** | c.1859G>A |
| **Protein** | p.Gly620Asp |
| **Genomic** | chr2:g.188974589C>T (GRCh38) |
| **Consequence** | Missense variant |
| **Exon** | 26 of 52 |
| **dbSNP** | rs387906622 |

## ACMG Classification: **PATHOGENIC** (Class 5)

### Criteria Applied

| Criterion | Strength | Evidence |
|-----------|----------|----------|
| **EDS_GlyXY** | Very Strong | Glycine→Aspartate substitution at position 620 in the Gly-X-Y repeat of the COL3A1 triple helical domain (residues 168–1196). Glycine is the only amino acid small enough for the sterically restricted interior of the collagen triple helix. This substitution exerts a dominant-negative effect by disrupting triple helix folding. Position 620 is in the C-terminal half of the triple helix, associated with more severe vascular phenotype. |
| **PS1** | Strong | This exact amino acid change (p.Gly620Asp) has been previously reported as pathogenic in ClinVar by multiple submitters with no conflicts (review status: criteria provided, multiple submitters). |
| **PM1** | Moderate | Located in the triple helical domain of COL3A1 (Pfam: Collagen triple helix repeat), a well-established mutational hotspot for pathogenic variants causing vEDS. |
| **PP3** | Strong | Multiple computational predictors concordantly predict deleterious effect: REVEL=0.968 (≥0.932 → PP3_strong); CADD Phred=33.0 (≥25.3 → PP3_strong); AlphaMissense=0.987 (likely pathogenic). |
| **PM2** | Supporting | Absent from gnomAD population database (AF=0.0). Applied at supporting level per ClinGen SVI recommendation. |

**ACMG rule satisfied:** 1 Very Strong (EDS_GlyXY) + 1 Strong (PS1) + 1 Moderate (PM1) + 1 Strong (PP3) + 1 Supporting (PM2) → **Pathogenic** (exceeds threshold of 1 Very Strong + ≥1 Strong)

## Evidence Summary

### Population Data
- **gnomAD v4:** Not observed in any population (AF = 0). Consistent with a
  highly penetrant dominant condition — vEDS has a prevalence of ~1:50,000
  to 1:200,000.

### Computational Predictions
- **REVEL:** 0.968 — Strong evidence for pathogenicity (ClinGen threshold ≥0.932)
- **CADD Phred:** 33.0 — Strong evidence (≥25.3)
- **AlphaMissense:** 0.987 — Classified as "likely pathogenic"
- All predictors concordantly support deleteriousness.

### Protein Structure / Function
- Position 620 falls within the obligatory Gly-X-Y repeat region of the triple
  helical domain (residues 168–1196). The substitution of glycine with aspartate
  (a large, charged amino acid) prevents normal triple helix assembly.
- Dominant-negative mechanism: the mutant α1(III) chain is incorporated into
  procollagen trimers but disrupts folding, leading to overmodification and
  reduced secretion of all type III collagen.
- C-terminal position in the triple helix — glycine substitutions in this region
  are associated with earlier onset and more severe arterial complications.

### Clinical Data (ClinVar)
- **Clinical significance:** Pathogenic/Likely pathogenic
- **Review status:** criteria provided, multiple submitters, no conflicts
- **Condition:** Ehlers-Danlos syndrome, vascular type (OMIM #130050)

## Clinical Significance

This variant is classified as **Pathogenic** for **vascular Ehlers-Danlos Syndrome
(vEDS)**. vEDS is an autosomal dominant condition characterized by:
- Arterial rupture (typically medium-to-large arteries)
- Bowel perforation (especially sigmoid colon)
- Uterine rupture during pregnancy
- Characteristic facial features (thin nose, small lips, hollow cheeks)
- Thin, translucent skin with visible veins

**Management implications:** Patients with confirmed vEDS should be referred to a
connective tissue disorders center. Surveillance includes regular vascular imaging.
Celiprolol has shown benefit in reducing vascular events. Avoid invasive vascular
procedures when possible; minimize contact sports.

## Limitations
- This is a computational pre-classification and must be reviewed by a board-certified
  clinical geneticist before diagnostic reporting.
- Segregation data and de novo status were not available for this analysis.
- Functional studies specific to this exact variant were not identified in the
  literature search.

## References
1. Pepin MG, Schwarze U, et al. (2014) Molecular diagnosis in Vascular Ehlers-Danlos
   Syndrome Predicts Pattern of Arterial Involvement and Outcomes. *Genet Med*
   16:881-888. PMID: 24599722
2. Bowen CJ, Giadrossi JJ, et al. (2021) Four decades in the making: Collagen III
   and mechanisms of vascular Ehlers Danlos Syndrome. *Matrix Biol Plus* 12:100090.
   PMID: 34773761
3. Richards S, Aziz N, et al. (2015) Standards and Guidelines for the Interpretation
   of Sequence Variants. *Genet Med* 17:405-424. PMID: 25741868
4. Pejaver V, Byrne AB, et al. (2022) Calibration of computational tools for missense
   variant pathogenicity classification and ClinGen recommendations for PP3/BP4
   criteria. *Am J Hum Genet* 109:2163-2177. PMID: 36413997
5. Malfait F, Francomano C, et al. (2017) The 2017 international classification of
   the Ehlers-Danlos syndromes. *Am J Med Genet C* 175:8-26. PMID: 28306229

```

---

## 6. RAG Pattern for Genomics Literature

### Building a Knowledge Base of EDS Research Papers

Use Amazon Bedrock Knowledge Bases with OpenSearch Serverless as the vector store to create a searchable corpus of EDS-relevant literature.

Step 1: Curate the Literature Corpus

```python
"""Download and prepare EDS research papers for Knowledge Base ingestion."""

import requests
import json
import os

# Target journals and search terms for EDS literature
PUBMED_QUERIES = [
    '"vascular Ehlers-Danlos" OR "EDS type IV" OR "COL3A1"',
    '"Ehlers-Danlos syndrome" AND "variant classification"',
    '"collagen type III" AND "pathogenic variant"',
    '"Ehlers-Danlos" AND "ACMG" AND classification',
    '"COL5A1" OR "COL5A2" AND "Ehlers-Danlos"',
    '"TNXB" AND "Ehlers-Danlos"',
    '"COL1A1" OR "COL1A2" AND "Ehlers-Danlos"',
    'glycine substitution collagen triple helix pathogenicity',
]

# Fetch PMIDs and metadata
EUTILS = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils"

all_pmids = set()
for query in PUBMED_QUERIES:
    resp = requests.get(f"{EUTILS}/esearch.fcgi", params={
        "db": "pubmed",
        "term": query,
        "retmax": 200,
        "retmode": "json",
    })
    data = resp.json()
    pmids = data["esearchresult"]["idlist"]
    all_pmids.update(pmids)
    print(f"Query: {query[:50]}... → {len(pmids)} articles")

print(f"\nTotal unique PMIDs: {len(all_pmids)}")

# Download full-text PDFs where available (PubMed Central)
# Save to S3 for Knowledge Base ingestion

```

Step 2: Prepare Documents for Ingestion

```bash
# Upload to S3 data source bucket
aws s3 sync ./eds-literature/ \
  s3://eds-knowledge-base-${AWS_ACCOUNT_ID}/papers/ \
  --exclude "*.py" \
  --include "*.pdf" \
  --include "*.txt" \
  --include "*.json"

# Also include structured data:
# - ACMG guidelines (PDF)
# - ClinGen expert panel specifications
# - Gene-specific interpretation rules
# - EDS diagnostic criteria (2017 nosology)

```

Step 3: Create the Knowledge Base

```python
"""Create Bedrock Knowledge Base for EDS genomics literature."""

import boto3

bedrock_agent = boto3.client("bedrock-agent", region_name="us-east-1")

# Step 3a: Create OpenSearch Serverless collection
aoss_client = boto3.client("opensearchserverless", region_name="us-east-1")

# Create encryption policy
aoss_client.create_security_policy(
    name="eds-kb-encryption",
    type="encryption",
    policy=json.dumps({
        "Rules": [{"ResourceType": "collection", "Resource": ["collection/eds-genomics-kb"]}],
        "AWSOwnedKey": True,
    }),
)

# Create network policy
aoss_client.create_security_policy(
    name="eds-kb-network",
    type="network",
    policy=json.dumps([{
        "Rules": [{"ResourceType": "collection", "Resource": ["collection/eds-genomics-kb"]}],
        "AllowFromPublic": True,  # Restrict in production
    }]),
)

# Create collection
collection = aoss_client.create_collection(
    name="eds-genomics-kb",
    type="VECTORSEARCH",
    description="EDS genomics literature vector store",
)

# Step 3b: Create Knowledge Base
kb_response = bedrock_agent.create_knowledge_base(
    name="eds-variant-interpretation-kb",
    description="EDS research literature, ACMG guidelines, and gene-specific interpretation rules",
    roleArn=f"arn:aws:iam::{account_id}:role/BedrockKBRole",
    knowledgeBaseConfiguration={
        "type": "VECTOR",
        "vectorKnowledgeBaseConfiguration": {
            "embeddingModelArn": f"arn:aws:bedrock:us-east-1::foundation-model/amazon.titan-embed-text-v2:0",
            "embeddingModelConfiguration": {
                "bedrockEmbeddingModelConfiguration": {
                    "dimensions": 1024,
                }
            },
        },
    },
    storageConfiguration={
        "type": "OPENSEARCH_SERVERLESS",
        "opensearchServerlessConfiguration": {
            "collectionArn": collection["createCollectionDetail"]["arn"],
            "vectorIndexName": "eds-papers-index",
            "fieldMapping": {
                "vectorField": "embedding",
                "textField": "text",
                "metadataField": "metadata",
            },
        },
    },
)

kb_id = kb_response["knowledgeBase"]["knowledgeBaseId"]
print(f"Knowledge Base created: {kb_id}")

# Step 3c: Add S3 data source
ds_response = bedrock_agent.create_data_source(
    knowledgeBaseId=kb_id,
    name="eds-papers",
    dataSourceConfiguration={
        "type": "S3",
        "s3Configuration": {
            "bucketArn": f"arn:aws:s3:::eds-knowledge-base-{account_id}",
            "inclusionPrefixes": ["papers/"],
        },
    },
    vectorIngestionConfiguration={
        "chunkingConfiguration": {
            "chunkingStrategy": "SEMANTIC",
            "semanticChunkingConfiguration": {
                "maxTokens": 512,
                "bufferSize": 0,
                "breakpointPercentileThreshold": 95,
            },
        },
    },
)

# Step 3d: Start ingestion
bedrock_agent.start_ingestion_job(
    knowledgeBaseId=kb_id,
    dataSourceId=ds_response["dataSource"]["dataSourceId"],
)

```

### Embedding Strategy for Scientific Text

| Strategy | Why |
| --- | --- |
| **Semantic chunking** (not fixed-size) | Scientific papers have logical sections; semantic boundaries preserve context better than arbitrary 500-token splits |
| **512 tokens max per chunk** | Balances between capturing enough context and maintaining embedding specificity |
| **Titan Embeddings v2 (1024-dim)** | AWS-native, excellent performance on scientific text; 1024 dims provides good accuracy without excessive storage |
| **Metadata enrichment** | Attach PMID, gene names, variant IDs, publication year to each chunk for filtered retrieval |

### Retrieval and Citation in Variant Reports

```python
"""RAG tool for EDS literature search."""

from strands import tool
import boto3

bedrock_runtime = boto3.client("bedrock-agent-runtime", region_name="us-east-1")


@tool
def search_eds_literature(
    query: str,
    gene: str = "",
    max_results: int = 5,
) -> dict:
    """Search the EDS genomics knowledge base for relevant literature and guidelines.

    This searches a curated corpus of EDS research papers, ACMG guidelines,
    and gene-specific interpretation rules using semantic search.

    Args:
        query: Natural language search query describing the information needed.
        gene: Optional gene symbol to filter results.
        max_results: Maximum number of results (default 5).

    Returns:
        Dictionary with relevant text passages, source citations (PMIDs),
        and relevance scores.
    """
    # Add gene context to query
    full_query = query
    if gene:
        full_query = f"{gene}: {query}"

    response = bedrock_runtime.retrieve(
        knowledgeBaseId=KB_ID,
        retrievalQuery={"text": full_query},
        retrievalConfiguration={
            "vectorSearchConfiguration": {
                "numberOfResults": max_results,
                "overrideSearchType": "HYBRID",  # Combines semantic + keyword
            }
        },
    )

    results = []
    for item in response.get("retrievalResults", []):
        content = item.get("content", {}).get("text", "")
        location = item.get("location", {})
        score = item.get("score", 0)

        # Extract source metadata
        source_uri = location.get("s3Location", {}).get("uri", "")
        metadata = item.get("metadata", {})

        results.append({
            "text": content,
            "score": score,
            "source": source_uri,
            "pmid": metadata.get("pmid", ""),
            "title": metadata.get("title", ""),
            "year": metadata.get("year", ""),
        })

    return {
        "found": len(results) > 0,
        "query": full_query,
        "results": results,
    }

```

---

## 7. Deployment & Cost Estimation

### CDK Stack for Core Infrastructure

```typescript
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';
import * as iam from 'aws-cdk-lib/aws-iam';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as events from 'aws-cdk-lib/aws-events';
import * as targets from 'aws-cdk-lib/aws-events-targets';
import * as athena from 'aws-cdk-lib/aws-athena';
import * as opensearchserverless from 'aws-cdk-lib/aws-opensearchserverless';
import { Construct } from 'constructs';

export class GenomicVariantAgentStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // ─── S3 Buckets ───────────────────────────────────────────
    const dataBucket = new s3.Bucket(this, 'GenomicsDataBucket', {
      bucketName: `genomics-variant-store-${this.account}`,
      encryption: s3.BucketEncryption.S3_MANAGED,
      versioned: true,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
    });

    const athenaResultsBucket = new s3.Bucket(this, 'AthenaResultsBucket', {
      bucketName: `genomics-athena-results-${this.account}`,
      lifecycleRules: [{
        expiration: cdk.Duration.days(30),  // Auto-cleanup query results
      }],
    });

    // ─── IAM Roles ────────────────────────────────────────────
    // HealthOmics Workflow Role
    const omicsWorkflowRole = new iam.Role(this, 'OmicsWorkflowRole', {
      assumedBy: new iam.ServicePrincipal('omics.amazonaws.com'),
      description: 'Role for HealthOmics workflows to access S3 and ECR',
    });
    dataBucket.grantReadWrite(omicsWorkflowRole);

    // AgentCore Harness Execution Role
    const harnessRole = new iam.Role(this, 'AgentCoreHarnessRole', {
      assumedBy: new iam.ServicePrincipal('bedrock-agentcore.amazonaws.com'),
      description: 'Execution role for AgentCore variant interpreter harness',
    });

    harnessRole.addToPolicy(new iam.PolicyStatement({
      actions: [
        'bedrock:InvokeModel',
        'bedrock:InvokeModelWithResponseStream',
      ],
      resources: [
        `arn:aws:bedrock:${this.region}::foundation-model/anthropic.claude-sonnet-4-5-20250514-v1:0`,
        `arn:aws:bedrock:${this.region}::foundation-model/us.anthropic.claude-sonnet-4-5-20250514-v1:0`,
      ],
    }));

    harnessRole.addToPolicy(new iam.PolicyStatement({
      actions: [
        'athena:StartQueryExecution',
        'athena:GetQueryExecution',
        'athena:GetQueryResults',
      ],
      resources: ['*'],
    }));

    harnessRole.addToPolicy(new iam.PolicyStatement({
      actions: [
        'glue:GetDatabase',
        'glue:GetTable',
        'glue:GetPartitions',
      ],
      resources: [
        `arn:aws:glue:${this.region}:${this.account}:catalog`,
        `arn:aws:glue:${this.region}:${this.account}:database/genomics_db`,
        `arn:aws:glue:${this.region}:${this.account}:table/genomics_db/*`,
      ],
    }));

    dataBucket.grantRead(harnessRole);
    athenaResultsBucket.grantReadWrite(harnessRole);

    // Knowledge Base role
    harnessRole.addToPolicy(new iam.PolicyStatement({
      actions: [
        'bedrock:Retrieve',
        'bedrock:RetrieveAndGenerate',
      ],
      resources: [`arn:aws:bedrock:${this.region}:${this.account}:knowledge-base/*`],
    }));

    // ─── Athena Workgroup ─────────────────────────────────────
    new athena.CfnWorkGroup(this, 'GenomicsWorkgroup', {
      name: 'genomics-variant-analysis',
      workGroupConfiguration: {
        resultConfiguration: {
          outputLocation: `s3://${athenaResultsBucket.bucketName}/results/`,
          encryptionConfiguration: {
            encryptionOption: 'SSE_S3',
          },
        },
        engineVersion: {
          selectedEngineVersion: 'Athena engine version 3',
        },
        bytesScannedCutoffPerQuery: 10_000_000_000,  // 10 GB limit
      },
    });

    // ─── EventBridge Rule for VEP Workflow Completion ─────────
    const workflowCompleteRule = new events.Rule(this, 'VEPWorkflowComplete', {
      eventPattern: {
        source: ['aws.omics'],
        detailType: ['Run Status Change'],
        detail: {
          status: ['COMPLETED'],
        },
      },
    });

    // Lambda to trigger Iceberg table loading
    const icebergLoaderFn = new lambda.Function(this, 'IcebergLoader', {
      runtime: lambda.Runtime.PYTHON_3_12,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/iceberg-loader'),
      timeout: cdk.Duration.minutes(15),
      memorySize: 2048,
      environment: {
        DATA_BUCKET: dataBucket.bucketName,
        ATHENA_DATABASE: 'genomics_db',
      },
    });
    dataBucket.grantReadWrite(icebergLoaderFn);
    workflowCompleteRule.addTarget(new targets.LambdaFunction(icebergLoaderFn));

    // ─── Outputs ──────────────────────────────────────────────
    new cdk.CfnOutput(this, 'DataBucketName', {
      value: dataBucket.bucketName,
    });
    new cdk.CfnOutput(this, 'HarnessRoleArn', {
      value: harnessRole.roleArn,
    });
  }
}

```

### Deploy the AgentCore Harness

```bash
# Install AgentCore CLI
pip install bedrock-agentcore-cli

# Initialize project
agentcore init --name genomic-variant-interpreter

# Create harness
agentcore add harness \
  --name variant-interpreter \
  --model-provider bedrock \
  --model-id us.anthropic.claude-sonnet-4-5-20250514-v1:0 \
  --system-prompt "$(cat system_prompt.txt)"

# Add tools (inline functions — executed client-side)
agentcore add tool --harness variant-interpreter --type inline_function \
  --name query_annotation_store \
  --description "Query ClinVar/gnomAD annotations via Athena" \
  --input-schema "$(cat schemas/annotation_store.json)"

agentcore add tool --harness variant-interpreter --type inline_function \
  --name query_vep \
  --description "Query Ensembl VEP REST API" \
  --input-schema "$(cat schemas/vep.json)"

agentcore add tool --harness variant-interpreter --type inline_function \
  --name search_pubmed \
  --description "Search PubMed for relevant literature" \
  --input-schema "$(cat schemas/pubmed.json)"

agentcore add tool --harness variant-interpreter --type inline_function \
  --name apply_acmg_criteria \
  --description "Apply ACMG/AMP classification criteria" \
  --input-schema "$(cat schemas/acmg.json)"

# Deploy
agentcore deploy

# Test
agentcore invoke --harness variant-interpreter \
  "Interpret COL3A1 c.1859G>A (p.Gly620Asp) for vascular EDS"

```

### Cost Estimation Per Variant Interpretation

| Component | Per Invocation | Notes |
| --- | --- | --- |
| **Bedrock (Claude Sonnet 4)** | ~$0.05 – $0.15 | ~3K input tokens (system prompt + tool schemas) + ~2K per tool call (×4 tools) + ~4K output tokens. At $3/$15 per 1M input/output tokens |
| **AgentCore Runtime** | ~$0.006 | Serverless microVM, ~30 sec active session |
| **Athena Query** | ~$0.005 | ~1 GB scanned for variant lookup ($5/TB) |
| **S3 Tables** | ~$0.001 | Storage: $0.023/GB/month; requests negligible |
| **Ensembl VEP API** | Free | REST API, rate-limited to 15 req/sec |
| **PubMed E-utilities** | Free | Rate-limited to 3 req/sec (10 with API key) |
| **Bedrock KB Retrieval** | ~$0.002 | $0.02 per 1000 retrieval units |
| **OpenSearch Serverless** | ~$0.01 (amortized) | 2 OCU minimum ($0.24/OCU/hr) amortized across queries |
| **Total per interpretation** | **~$0.07 – $0.20** | Without RAG: ~$0.07; With RAG: ~$0.15–$0.20 |

### Scaling Considerations

| Scale | Architecture | Notes |
| --- | --- | --- |
| **1–50 variants/day** | Single Strands agent, direct Athena queries | Simplest; no additional infrastructure needed |
| **50–500 variants/day** | AgentCore Harness with inline functions; Athena query caching | Consider Athena result caching; batch VEP calls via POST endpoint |
| **500–5,000 variants/day** | Multi-agent: Orchestrator + parallel tool workers; DynamoDB for result caching | Pre-compute frequent variants; cache VEP results in DynamoDB |
| **5,000+ variants/day** | Batch processing pipeline: Step Functions → Batch → Agent for novel variants only | Pre-annotate entire VCF with VEP batch; agent only for novel/ambiguous variants |

VEP API Rate Limits

```
┌──────────────────────────────────────────────────────────┐
│  Ensembl VEP REST API Rate Limits                        │
├──────────────────────────────────────────────────────────┤
│  GET endpoint:  15 requests/second                       │
│  POST endpoint: 15 requests/second, 200 variants/request │
│                                                          │
│  For bulk processing, use POST /vep/human/hgvs:          │
│  curl -X POST https://rest.ensembl.org/vep/human/hgvs \  │
│    -H "Content-Type: application/json" \                  │
│    -d '{"hgvs_notations": ["...", "..."]}'               │
│                                                          │
│  Max throughput: 3,000 variants/second via POST batch    │
│                                                          │
│  For >10K variants/day: consider local VEP installation  │
│  via HealthOmics Workflows (Docker container on Batch)   │
└──────────────────────────────────────────────────────────┘

```

---

## Appendix A: Key References

1. **Richards S, et al.** (2015) Standards and Guidelines for the Interpretation of Sequence Variants: A Joint Consensus Recommendation of ACMG and AMP. *Genet Med* 17:405–424. [PMID: 25741868](https://pubmed.ncbi.nlm.nih.gov/25741868/)
2. **Pejaver V, et al.** (2022) Calibration of computational tools for missense variant pathogenicity classification and ClinGen recommendations for PP3/BP4 criteria. *Am J Hum Genet* 109:2163–2177. [PMID: 36413997](https://pubmed.ncbi.nlm.nih.gov/36413997/)
3. **Pepin MG, et al.** (2014) Molecular diagnosis in Vascular Ehlers-Danlos Syndrome Predicts Pattern of Arterial Involvement and Outcomes. *Genet Med* 16:881–888. [PMID: 24599722](https://pubmed.ncbi.nlm.nih.gov/24599722/)
4. **Malfait F, et al.** (2017) The 2017 international classification of the Ehlers-Danlos syndromes. *Am J Med Genet C* 175:8–26. [PMID: 28306229](https://pubmed.ncbi.nlm.nih.gov/28306229/)
5. **Bowen CJ, et al.** (2021) Four decades in the making: Collagen III and mechanisms of vascular Ehlers Danlos Syndrome. *Matrix Biol Plus* 12:100090. [PMID: 34773761](https://pubmed.ncbi.nlm.nih.gov/34773761/)
6. **AWS Blog:** Sandanaraj E, Lee C, Poonawala H. (2025) [Accelerating genomics variant interpretation with AWS HealthOmics and Amazon Bedrock AgentCore](https://aws.amazon.com/blogs/machine-learning/accelerating-genomics-variant-interpretation-with-aws-healthomics-and-amazon-bedrock-agentcore/). AWS Machine Learning Blog.
7. **Ensembl VEP REST API:** [GET vep/:species/hgvs/:hgvs_notation](https://rest.ensembl.org/documentation/info/vep_hgvs_get)
8. **Strands Agents SDK:** [@tool decorator documentation](https://strandsagents.com/docs/api/python/strands.tools.decorator/)

## Appendix B: Glossary for the AWS SA Learning Genomics

| Term | Definition |
| --- | --- |
| **HGVS notation** | Standard nomenclature for variant naming: `c.` = coding DNA, `p.` = protein, `g.` = genomic |
| **Gly-X-Y repeat** | Repeating tripeptide in collagen where every 3rd position must be glycine |
| **Triple helix** | The fundamental structural unit of collagen — three polypeptide chains wound around each other |
| **Dominant-negative** | Mutant protein interferes with normal protein function (worse than loss of one copy) |
| **Haploinsufficiency** | Disease caused by having only one functional gene copy (50% protein) |
| **gnomAD** | Genome Aggregation Database — population allele frequency reference (~800K exomes/genomes) |
| **ClinVar** | NCBI database of clinically interpreted genetic variants |
| **VEP** | Variant Effect Predictor — Ensembl tool for predicting functional consequences of variants |
| **CADD** | Combined Annotation Dependent Depletion — integrative deleteriousness score |
| **REVEL** | Rare Exome Variant Ensemble Learner — ensemble pathogenicity predictor for missense variants |
| **SpliceAI** | Deep learning model (Illumina) that predicts splice junction effects |
| **AlphaMissense** | DeepMind model predicting missense variant pathogenicity from protein structure |
| **vEDS** | Vascular Ehlers-Danlos Syndrome (EDS type IV) — the most severe EDS subtype |
| **ACMG** | American College of Medical Genetics and Genomics |
| **VUS** | Variant of Uncertain Significance — insufficient evidence to classify as pathogenic or benign |

