# DocFlow — Serverless Document Intelligence on AWS

Event-driven pipeline that turns unstructured documents (invoices, bank statements, financial
statements) into structured, validated JSON — written entirely in **Terraform**.

Drop a PDF into S3; EventBridge and SQS queue it, a Step Functions workflow runs **Textract** OCR,
then **Amazon Bedrock** models classify the document, extract schema-defined fields, score their own
confidence and summarise it. Every job is tracked in DynamoDB and watched from a CloudWatch dashboard.

## Architecture

```
S3 (input) ──► EventBridge ──► Lambda: queue sender ──► SQS (+ DLQ)
                                      │                     │
                                      ▼                     ▼
                              DynamoDB: tracking ◄── Lambda: queue processor
                              + concurrency counter         │ StartExecution
                                                            ▼
                                                  Step Functions (Standard)
          ┌──────────┬──────────────┬──────────────┬──────────────┬──────────────┐
          ▼          ▼              ▼              ▼              ▼
       Textract   Bedrock        Bedrock        Bedrock        Bedrock
         OCR      classify       extract        assess        summarise ──► S3 (output)

                  CloudWatch: dashboard · logs · alarms
```

| Service | Role |
|---|---|
| **S3** | Input and output buckets (KMS-encrypted, public access blocked) |
| **EventBridge** | Routes `Object Created` events to the queue sender |
| **SQS + DLQ** | Buffers bursts, retries, isolates poison messages |
| **DynamoDB** | Job tracking, configuration, and an atomic concurrency counter that keeps Bedrock/Textract under quota |
| **Step Functions** | Orchestrates the pipeline with per-step retries and backoff |
| **Lambda** | Queue handling and each processing step |
| **Textract** | OCR with layout, tables and forms |
| **Bedrock** | Classification, extraction, confidence assessment, summarisation (Claude / Amazon Nova) |
| **CloudWatch** | Throughput, latency and failure dashboards, logs, alarms |

## Deploy

Prerequisites: Terraform ≥ 1.5, AWS CLI credentials, Bedrock model access enabled in your region.

```bash
git clone --recursive https://github.com/Sayandeep1013/DocFlow-AWS.git
cd DocFlow-AWS/examples/bedrock-llm-processor
cp terraform.tfvars.example terraform.tfvars   # region, prefix, model IDs
terraform init
terraform plan -out tf.plan
terraform apply tf.plan
```

Upload a PDF to the input bucket printed in the outputs, then watch the execution in the
Step Functions console. Tear everything down with `terraform destroy`.

> `sources/` is a git submodule holding the Lambda source code. Clone with `--recursive`
> or run `git submodule update --init`.

## CI

| Workflow | Checks |
|---|---|
| `terraform-validation.yml` | `terraform fmt -check`, `terraform validate` per module |
| `iac-security.yml` | **tflint**, **tfsec** (Trivy config scan), **checkov** |

## Repository layout

```
main.tf, variables.tf, …   root module
modules/                   processing environment, processors, API, web UI, knowledge base
examples/                  deployable stacks — start with bedrock-llm-processor/
sources/                   Lambda source (git submodule)
docs/                      design docs and migration notes
```

## Roadmap

- [ ] Deterministic ratio step: compute financial ratios in code from extracted values, never in the LLM
- [ ] Custom document classes for Indian financial documents (GST invoices, bank statements)
- [ ] Remote Terraform state (S3 backend with locking)

## Credits and licence

Built on the Terraform edition of AWS's GenAI Intelligent Document Processing Accelerator
([awslabs/genai-idp-terraform](https://github.com/awslabs/genai-idp-terraform)) and the
underlying [IDP solution](https://github.com/aws-solutions-library-samples/accelerated-intelligent-document-processing-on-aws).
Licensed under Apache-2.0 — see [LICENSE](LICENSE) and [NOTICE](NOTICE).
