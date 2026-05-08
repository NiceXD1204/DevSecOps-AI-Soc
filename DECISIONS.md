# Architecture Decision Records (ADR)

This document captures the key technical and architectural decisions 
made during the development of the AI-Powered Hybrid DevSecOps SOC 
project. Each decision includes context, alternatives considered, 
trade-offs, and final rationale.

---

## ADR-001: AWS Region Selection

**Decision:** eu-central-1 (Frankfurt)

**Context:** Need to choose primary AWS region for hosting the project 
infrastructure (EC2, VPC, S3 backend).

**Alternatives Considered:**
- us-east-1 (N. Virginia): cheapest pricing, most tutorials, but ~180ms 
  latency from Israel
- il-central-1 (Tel Aviv): lowest latency, but newer region with fewer 
  services and higher pricing
- eu-west-1 (Ireland): similar pricing to Frankfurt, slightly higher 
  latency

**Rationale:**
- Geographic proximity: ~70ms latency from Israel enables responsive 
  interactive work (SSH, kubectl, terraform apply)
- GDPR compliance: enables potential future expansion to EU enterprise 
  customers without data residency concerns
- Aligns with project's privacy-first narrative (local AI inference, 
  no external API calls for security data)

**Trade-offs Accepted:**
- ~10% higher pricing than us-east-1 (negligible at project scale)
- Some AWS services launch later in eu-central-1
- Most tutorials assume us-east-1 (mitigated by Terraform's 
  region-agnostic design)

---

## ADR-002: AI Model Strategy - Local Inference

**Decision:** Run Gemma 4 E4B locally via Ollama on workstation, not 
in cloud or via external API.

**Context:** The system needs to analyze Falco security events using 
an LLM and recommend remediation actions.

**Alternatives Considered:**
- OpenAI/Anthropic API: high quality but sends sensitive security 
  events to third-party servers
- Cloud-hosted LLM on EC2: requires GPU instance ($500+/month) or 
  CPU-only inference (too slow)
- OpenMythos: theoretical research model, requires training from 
  scratch (impractical)

**Rationale:**
- Data Sovereignty: security events never leave the organization 
  perimeter, addressing concerns of regulated industries (finance, 
  healthcare, defense)
- Cost: zero inference cost vs. $0.01-0.10 per analyzed event with 
  external APIs
- Demonstrates production-grade thinking: real enterprises in regulated 
  sectors cannot send security telemetry to external AI providers
- Workstation specs (16GB VRAM) sufficient for E4B model with GPU 
  acceleration

**Trade-offs Accepted:**
- Workstation must be online during demos
- Slightly lower analysis quality vs. frontier API models
- More complex networking (SSH tunnel for event delivery)

---

## ADR-003: Hybrid Cloud-Local Architecture

**Decision:** Two-tier architecture with Workload Layer in AWS and 
Analysis Layer on local workstation, connected via Zero Trust SSH 
tunnel.

**Context:** Need to balance "production-realistic" cloud deployment 
with cost constraints and the local AI requirement (ADR-002).

**Architecture:**
- AWS EC2 (t3.large): k3s cluster, vulnerable application, Falco, 
  Prometheus, Grafana, Falcosidekick (localhost-only listener)
- Local Workstation: Ollama, Gemma 4 E4B, Python AI Analyzer, 
  Human-in-the-Loop approval UI
- Connection: SSH tunnel initiated from workstation to EC2 (no inbound 
  ports exposed beyond SSH)

**Alternatives Considered:**
- Everything in AWS with cloud GPU: $500+/month, contradicts data 
  sovereignty story
- Everything local with minikube: doesn't demonstrate cloud/Terraform 
  skills required for DevOps roles
- AI in management cluster (separate EC2): adds cost and complexity 
  without clear benefit at portfolio scale

**Rationale:**
- Mirrors real-world enterprise patterns: edge sensors send events to 
  centralized analysis layer
- Demonstrates both DevOps (cloud infrastructure) and security 
  (architecture-aware) skills
- SSH tunnel pattern is industry-standard for secure cross-network 
  communication
- Cost-efficient: workstation idle resources used productively

---

## ADR-004: Human-in-the-Loop for AI Recommendations

**Decision:** AI generates remediation recommendations that require 
explicit human approval before execution. No autonomous blocking.

**Context:** Initial design considered fully autonomous AI-driven 
remediation (LLM generates and executes kubectl commands).

**Alternatives Considered:**
- Fully autonomous: LLM analyzes Falco event, generates kubectl 
  command, executes immediately
- Read-only AI: AI only describes what happened, no action 
  recommendations

**Rationale:**
- Security best practice: LLM-generated commands can hallucinate or 
  be exploited via prompt injection in attack payloads
- Reflects mature production patterns: SOAR (Security Orchestration, 
  Automation, Response) systems typically require analyst approval 
  for destructive actions
- Builds trust: humans verify the AI's reasoning before acting
- Educational value: demonstrates understanding of AI safety, not just 
  AI capability

**Trade-offs Accepted:**
- Slower response time during attacks (human approval introduces delay)
- Requires human availability during operations

---

## ADR-005: Project Phasing - MVP First Approach

**Decision:** Project structured as MVP (stages 0-4) followed by 
Extension (stages 5-6), enabling job applications mid-project.

**Context:** Full project (with AI integration) estimated at 9-10 
weeks. Goal is to enter job market as early as possible.

**Phasing:**
- MVP (6-7 weeks): Cloud infrastructure, Kubernetes, CI/CD, Observability
- Extension (3 weeks): Security detection layer + AI integration

**Rationale:**
- After MVP, portfolio includes core DevOps stack (Terraform, k3s, 
  GitHub Actions, ArgoCD, Prometheus, Grafana) sufficient for entry/mid 
  level applications
- AI extension serves as differentiator developed in parallel with 
  interview process
- Reduces risk of over-investment before market validation
- Demonstrates iterative delivery (a DevOps practice)

---

## ADR-006: IAM Strategy - Least Privilege Custom Policy

**Decision:** Custom IAM policy for Terraform user, scoped to project 
needs, instead of AdministratorAccess.

**Context:** Terraform requires AWS credentials to provision 
infrastructure. Default tutorials suggest AdministratorAccess, but 
this violates principle of least privilege.

**Alternatives Considered:**
- AdministratorAccess: simplest but maximum blast radius if credentials 
  leak
- PowerUserAccess (AWS-managed): broad but excludes IAM, requires 
  supplementing
- Custom policy per service: most secure but high friction during 
  development

**Rationale:**
- Custom policy explicitly scopes permissions to project needs: EC2, 
  VPC, IAM (limited), S3, DynamoDB, CloudWatch, SSM, Budgets
- Critical restrictions:
  - No iam:CreateUser/CreateAccessKey (prevents persistence if leaked)
  - iam:PassRole scoped to roles matching "devsecops-*" prefix
  - iam:PassedToService condition limits target services to EC2/SSM
- Demonstrates security-first thinking in a DevOps context
- Documents exactly what infrastructure the project requires

**Trade-offs Accepted:**
- Some friction if adding new AWS services (requires policy update)
- More complex setup vs. AdministratorAccess

---

## ADR-007: AWS Authentication Method

**Decision:** Long-lived Access Keys with planned MFA enforcement and 
AWS Vault for credential encryption.

**Context:** AWS recommends transitioning away from long-lived access 
keys to temporary credentials (aws sso login, IAM Identity Center).

**Alternatives Considered:**
- aws sso login: more secure, but adds friction during long Terraform 
  operations and requires IAM Identity Center setup
- AWS CloudShell: eliminates local credential risk but breaks IDE 
  workflow (VS Code, Git, local development)

**Rationale:**
- Industry standard: matches credentials patterns in most DevOps job 
  environments
- AWS Vault encrypts credentials in OS keyring (Windows Credential 
  Manager), mitigating disk-leak risk
- MFA adds second factor for sensitive operations
- Monthly key rotation policy reduces long-term exposure window
- git-secrets pre-commit hooks prevent accidental commits to GitHub

**Trade-offs Accepted:**
- Credentials persist on disk (mitigated by encryption)
- Manual rotation required
- Higher risk than fully temporary credentials

---

## ADR-008: Terraform State Backend - S3 with DynamoDB Locking

**Decision:** Remote state in S3 bucket with DynamoDB table for state 
locking, configured from project initialization.

**Context:** Terraform state file tracks all managed infrastructure. 
Default behavior stores state locally as terraform.tfstate.

**Alternatives Considered:**
- Local state file: simplest, free, but tied to single machine, 
  unsuitable for multi-machine workflow, no locking, risk of loss
- Terraform Cloud (free tier): free for up to 5 users, good UI, but 
  external dependency and not industry-standard
- HashiCorp Consul: too complex for project scale

**Rationale:**
- Industry standard: S3 backend is the most common Terraform state 
  pattern in production AWS environments
- Multi-machine portability: enables transition between development 
  machines (current vs. future workstation) without state loss
- Locking via DynamoDB: prevents concurrent terraform apply operations 
  from corrupting state
- Versioning enabled on S3 bucket: provides state history and recovery
- Encryption at rest: state contains sensitive infrastructure details

**Cost Analysis:**
- S3 storage: <$0.001/month for state file (~50KB)
- DynamoDB on-demand: $0.10-0.30/month for low-volume locking
- Within AWS Free Tier for first 12 months
- Total: $0.30/month or less, ~1.5% of total project cost

**Trade-offs Accepted:**
- Bootstrap chicken-and-egg: bucket and table must be created before 
  Terraform can use them as backend (one-time manual setup)
- Slightly more complex initialization

---

## ADR-009: Project Structure and Naming Conventions

**Decision:** Standard Terraform project structure with clear separation 
of concerns and consistent naming.

**Naming Convention:**
- All AWS resources prefixed with "devsecops-" for IAM policy compliance
  and easy identification
- Tags applied to all resources: Project, Environment, Owner, ManagedBy

**Repository Structure:**
devsecops-soc/
├── .gitignore           # Excludes state, credentials, IDE files
├── README.md            # Project overview, architecture, setup
├── DECISIONS.md         # This file - architectural decisions
├── iam-policies/        # Custom IAM policies as code
└── terraform/           # Infrastructure as Code
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
└── terraform.tfvars.example

**Rationale:**
- Standard structure: matches conventions used in production Terraform 
  codebases, aids reviewer comprehension
- Naming prefix enables IAM policy scoping and resource identification
- Tags enable cost tracking, ownership attribution, and automated 
  resource lifecycle management
- IAM policies stored as code enables version control and reproducibility

---

## ADR-010: FinOps Discipline - Daily Tear-Down Cycle

**Decision:** Run `terraform apply` at start of work session, 
`terraform destroy` at end. Avoid keeping infrastructure running 24/7.

**Context:** EC2 t3.large costs ~$0.17/hour ($120/month if running 
24/7). Project budget targets $15-20/month total.

**Operational Model:**
- Estimated work hours: 10-15 hours/week (60 hours/month)
- Compute cost at 60 hours: ~$10/month (EC2)
- Persistent costs: EBS volume, S3, DynamoDB (~$3/month combined)
- Total: ~$13-15/month

**Rationale:**
- Demonstrates FinOps awareness: real-world DevOps requires cost 
  consciousness
- Forces clean infrastructure-as-code: no "snowflake" servers with 
  manual configuration
- Reproducibility: ability to destroy and recreate proves automation 
  completeness
- Budget alignment: keeps within AWS credit allocation

**Risk Mitigation:**
- AWS Budgets alerts at $25, $40, $50
- Hard cap at $150 with automated action
- State stored in S3 (persists across destroy cycles)

---

## ADR-011: Workstation OS - Native Windows over WSL2

**Decision:** Run Ollama and Python AI Analyzer natively on Windows 
with GPU acceleration, not through WSL2.

**Context:** Initial architecture considered WSL2 as the development 
environment. Ollama supports both native Windows and Linux/WSL2.

**Alternatives Considered:**
- WSL2: provides Linux environment familiar from server administration
- Dual boot Linux: complete native Linux but requires reboot
- VM with Linux: resource-heavy, GPU passthrough complications

**Rationale:**
- Ollama Native Windows supports GPU acceleration directly via CUDA
- WSL2 GPU passthrough exists but adds complexity and overhead
- VRAM is critical resource; native access maximizes available memory
- Development tools (VS Code, Git) work seamlessly on Windows
- Eliminates a layer of abstraction for the AI inference path

**Trade-offs Accepted:**
- Some Linux-specific tools require Windows alternatives
- Path conventions differ from cloud Linux environment (mitigated by 
  consistent use of forward slashes in scripts)
