# CI/CD Pipelines with Azure DevOps — React App & EpicBook Multi-Stage Deployment

Two CI/CD projects built with Azure DevOps Pipelines, progressing from a simple 4-stage React deployment to a two-pipeline enterprise model using Terraform and Ansible.

---

## What is CI/CD?

**CI (Continuous Integration)** — every commit to main automatically triggers a build and test. Broken code is caught immediately before it reaches any environment.

**CD (Continuous Deployment)** — after the build and tests pass, the code is automatically deployed to a server. No manual steps required.

\`\`\`
Developer commits code
        │
        ▼
Azure DevOps detects commit
        │
        ▼
Pipeline runs automatically:
  Build → Test → Publish → Deploy
        │
        ▼
App is live on the server
\`\`\`

Without CI/CD, deployment is manual — someone has to SSH into a server, pull code, and restart services every time. CI/CD automates this completely.

---

## Project 1 — React App (Simple 4-Stage Pipeline)

A single pipeline that builds, tests, publishes, and deploys a React app to a VM running Nginx.

### How It Works

\`\`\`
Commit to main
      │
      ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Stage 1   │────►│   Stage 2   │────►│   Stage 3   │────►│   Stage 4   │
│    Build    │     │    Test     │     │   Publish   │     │   Deploy    │
│             │     │             │     │             │     │             │
│ npm install │     │ npm test    │     │ Inspect     │     │ SSH copy    │
│ npm run     │     │             │     │ artifact    │     │ to VM       │
│ build       │     │ Fails here  │     │             │     │             │
│             │     │ if broken   │     │             │     │ Restart     │
│ Saves /build│     │             │     │             │     │ Nginx       │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
\`\`\`

### Why deploy the /build folder instead of source code?

The React source code in src/ cannot run in a browser directly — it is JSX, modern JavaScript, and imports that browsers do not understand. npm run build compiles everything into plain HTML, CSS, and JavaScript that any browser can run. The /build folder is what gets deployed, not the source.

### Why use artifacts?

The build runs on one agent. The deploy runs on another. Artifacts are how the /build folder travels between them — it is saved after the Build stage and downloaded in the Deploy stage. Without artifacts, the deploy agent would have no files to copy.

### Pipeline file
react-app-cicd/azure-pipelines.yml

### Setup Steps
1. Import https://github.com/pravinmishraaws/my-react-app into Azure Repos
2. Provision a VM with Terraform and install Nginx with Ansible
3. Create an SSH Service Connection in Azure DevOps (ubuntu-nginx-ssh)
4. Create a new pipeline pointing to react-app-cicd/azure-pipelines.yml
5. Commit to main — pipeline triggers automatically

---

## Project 2 — EpicBook (Two-Pipeline Enterprise Model)

Two separate pipelines for two separate concerns: infrastructure and application. This is how teams operate in real production environments.

### Why Two Pipelines?

Infrastructure and application have different lifecycles:
- Infrastructure changes rarely — a new VM, a new subnet, a database scaling change
- Application deploys multiple times per day — every feature merge, every bug fix

Separating them means application engineers can deploy without touching infrastructure, and infrastructure changes can be reviewed independently.

\`\`\`
┌────────────────────────────────────────────────────┐
│                  INFRA PIPELINE                      │
│                                                      │
│  Terraform Init → Terraform Plan → Terraform Apply   │
│                                          │           │
│                                          ▼           │
│                              Outputs: app_public_ip  │
│                                        mysql_fqdn    │
└──────────────────────────────────────┬────────────────┘
                                     │
                          (pass IPs to App Pipeline)
                                     │
┌─────────────────────────────────────▼────────────────┐
│                   APP PIPELINE                       │
│                                                      │
│  Install Ansible                                     │
│  Download SSH key from Secure Files                  │
│  Generate inventory using Terraform outputs          │
│  Run Ansible playbook (common → nginx → epicbook)    │
│  Verify site is live                                 │
└────────────────────────────────────────────────────┘
\`\`\`

### How Terraform Outputs Feed Into Ansible

After terraform apply runs, the pipeline captures the VM's public IP and MySQL hostname:

\`\`\`bash
APP_IP=\$(terraform output -raw app_public_ip)
MYSQL_FQDN=\$(terraform output -raw mysql_fqdn)
\`\`\`

These are then injected into Ansible's inventory and group_vars before the playbook runs — so Ansible always knows exactly which server to configure and which database to connect to.

### Why Store SSH Keys in Secure Files?

SSH private keys must never be committed to git. Azure DevOps Secure Files is an encrypted store in your pipeline library. The key is downloaded at runtime into a temp location, used to connect to the VM, and never stored in the repo or pipeline logs.

### What is an Azure Resource Manager (SPN) Service Connection?

To run terraform apply, the pipeline needs permission to create resources in Azure. An SPN (Service Principal) is a non-human Azure AD identity with specific permissions. The pipeline authenticates as this identity using a Client ID and Client Secret stored in the service connection — never hardcoded in the pipeline YAML.

### Pipeline files
- epicbook-cicd/infra-pipeline/azure-pipelines.yml
- epicbook-cicd/app-pipeline/azure-pipelines.yml

### Setup Steps

**Infra Pipeline:**
1. Create an App Registration in Azure AD — note Client ID, Client Secret, Tenant ID, Subscription ID
2. Create an Azure Resource Manager service connection in Azure DevOps (azure-service-connection)
3. Create a Storage Account and container for Terraform remote state
4. Create a pipeline pointing to epicbook-cicd/infra-pipeline/azure-pipelines.yml
5. Run the pipeline — note app_public_ip and mysql_fqdn from the output

**App Pipeline:**
1. Upload your SSH private key to Azure DevOps Library → Secure Files (named id_ed25519)
2. Create a pipeline pointing to epicbook-cicd/app-pipeline/azure-pipelines.yml
3. Set pipeline variables: APP_PUBLIC_IP and MYSQL_FQDN from the infra pipeline output
4. Run the pipeline — Ansible configures the VM and deploys the app
5. Visit http://APP_PUBLIC_IP to confirm the site is live

---

## Project Structure

\`\`\`
├── react-app-cicd/
│   └── azure-pipelines.yml        # 4-stage pipeline: Build → Test → Publish → Deploy
│
├── epicbook-cicd/
│   ├── infra-pipeline/
│   │   └── azure-pipelines.yml    # Terraform: Init → Plan → Apply → Capture outputs
│   └── app-pipeline/
│       └── azure-pipelines.yml    # Ansible: Install → SSH key → Inventory → Playbook → Verify
│
└── README.md
\`\`\`

---

## Microsoft-Hosted vs Self-Hosted Agents

| | Microsoft-Hosted | Self-Hosted |
|--|-----------------|-------------|
| What it is | Azure spins up a fresh VM for each run | Your own VM registered as an agent |
| Cost | Free tier: 1 parallel job, 1800 mins/month | Free once VM exists |
| State | Clean every run — nothing persists | Persists between runs (caches, tools) |
| Use when | Standard builds, most CI/CD work | Faster builds (cached tools), private network access |

Both pipelines use vmImage: 'ubuntu-latest' (Microsoft-hosted). To use a self-hosted agent, replace with:
\`\`\`yaml
pool:
  name: 'SelfHostedPool'
\`\`\`

---

## Key Lessons Learned

| Problem | Root Cause | Fix |
|---------|-----------|-----|
| Pipeline not triggering on commit | Trigger branch name was master but repo used main | Update trigger to include: - main |
| Deploy stage ran even when tests failed | No dependsOn between stages | Add dependsOn: Test to each stage |
| SSH copy failed — permission denied on VM | /var/www/html owned by root | Pre-configure VM: sudo chown -R azureuser /var/www/html |
| Ansible inventory not found in pipeline | Working directory mismatch | Use full path: ansible/inventory.ini not just inventory.ini |
| SSH key download worked but Ansible still rejected it | Key permissions were 644 | Add chmod 600 \$(sshKey.secureFilePath) after download |
| Terraform apply failed — not authenticated | Service connection not linked to pipeline | Add service connection in pipeline settings under Security |
| Pipeline hung waiting for SSH host confirmation | First SSH connection prompts to accept host key | Add StrictHostKeyChecking=no to ansible_ssh_common_args |
| MySQL FQDN not passed to Ansible correctly | Variable had whitespace from terraform output | Use terraform output -raw (not just output) to strip quotes |

---

## What I Learned

**CI/CD removes human error from deployments** — every deployment follows the exact same steps in the exact same order. There is no "I forgot to restart Nginx" or "I deployed to the wrong server."

**Artifact publishing separates build from deploy** — building and deploying on the same agent is fragile. Publishing the /build folder as an artifact means the deploy stage is completely independent and can run on a different agent, at a different time, or be reused for other environments (staging, production).

**Two pipelines for two concerns** — infrastructure and application change at different rates and are owned by different people. A single monolithic pipeline would force application engineers to wait for infrastructure changes and vice versa.

**Secure Files for secrets** — SSH keys in git is a critical security vulnerability. Azure DevOps Secure Files encrypts the key and injects it at runtime. It never appears in the repo or in pipeline logs.

**StrictHostKeyChecking=no is needed in automation** — interactive SSH prompts block pipelines. In production you would pre-populate known_hosts instead, but disabling the check is acceptable for learning environments.

---

## Tools Used
Azure DevOps Pipelines, Azure Resource Manager (SPN), Terraform, Ansible, Nginx, Node.js 18, SSH, Ubuntu 22.04
