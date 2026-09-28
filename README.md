# AI-Powered Genomics for Ehlers-Danlos Syndrome Research

I'm on a mission to help my husband and my sons as my husband is diagnosed with Ehlers-Danlos Sundrom, and most likely my sons have it too. Therefore I  created a practical learning toolkit for applying AI/ML to genomics research, with a focus on Ehlers-Danlos Syndrome (EDS) — built for an AWS Solutions Architect with basic biology/chemistry background.

## 📁 Contents

| File | Description |
|------|-------------|
| `eds_genomics_study_plan.md` | **12-week study plan** — week-by-week learning path covering bioinformatics foundations → genomics pipelines → AWS HealthOmics → AI variant interpretation → ML classifier → advanced topics (Graph ML, federated learning). Includes verified URLs, key papers, datasets, and a capstone project blueprint. |
| `eds_variant_classifier_starter.py` | **Runnable Python script** — downloads real ClinVar data for 14 EDS genes, parses variants, engineers collagen-specific features (Gly-X-Y detection), and runs exploratory analysis. Includes TODO sketches for XGBoost training and SageMaker deployment. |
| `bedrock_variant_agent_guide.md` | **Technical guide** — architecture, code, and deployment instructions for building an LLM-powered variant interpretation agent using AWS Bedrock AgentCore + HealthOmics. Includes ACMG rules engine in Python, tool schemas, a real COL3A1 variant walkthrough, and CDK stack. |

## 🚀 Quick Start

```bash
# 1. Install dependencies
pip install pandas requests

# 2. Run the classifier starter (downloads ~100MB ClinVar data on first run)
python eds_variant_classifier_starter.py

# 3. Open the study plan and follow week-by-week
open eds_genomics_study_plan.md
```

## 🧬 Key EDS Genes Covered

| Gene | EDS Subtype | Role |
|------|-------------|------|
| COL5A1, COL5A2 | Classical | Type V collagen structure |
| COL3A1 | Vascular | Type III collagen (arterial integrity) |
| COL1A1, COL1A2 | Arthrochalasia | Type I collagen |
| TNXB | Classical-like | ECM regulatory glycoprotein |
| PLOD1, FKBP14 | Kyphoscoliotic | Collagen cross-linking enzymes |
| ADAMTS2 | Dermatosparaxis | Procollagen processing |
| COL12A1 | Myopathic | Type XII collagen |
| B4GALT7, B3GALT6, SLC39A13 | Spondylodysplastic | GAG synthesis |
| CHST14 | Musculocontractural | Dermatan sulfate biosynthesis |

## 🔬 Key References

- Shirvani et al. (2026) — *Multi-System Genetic Architecture of hEDS: Integrating ML with Subject-Level Genomic Analysis* — [MDPI Genes](https://www.mdpi.com/2073-4425/17/2/211)
- aiDIVA (2026) — *Hybrid AI for rare disease diagnostics* — [Nature](https://www.nature.com/articles/s41525-026-00611-x)
- GenPhenia (2026) — *Deep neural networks for rare disease diagnosis* — [HGG Advances](https://link.springer.com/article/10.1186/s40246-026-01023-9)
- AWS Blog (2025) — *Accelerating genomics variant interpretation with HealthOmics and Bedrock AgentCore* — [AWS ML Blog](https://aws.amazon.com/blogs/machine-learning/accelerating-genomics-variant-interpretation-with-aws-healthomics-and-amazon-bedrock-agentcore/)

## 🏗️ AWS Services Used

HealthOmics (Workflows, Stores) · SageMaker · Bedrock AgentCore · Athena · Neptune · S3 · Lambda · QuickSight · HealthLake · Clean Rooms

