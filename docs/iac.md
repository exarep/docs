# Infrastructure as Code

The `iac` project uses Ansible to provision and manage Exarep infrastructure on AWS. It handles DNS configuration across Cloudflare and Route 53, and deploys OpenShift clusters using the `openshift-install` IPI installer.

## Prerequisites

- Python 3
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) configured with valid credentials
- [openshift-install](https://console.redhat.com/openshift/install) CLI for cluster provisioning
- A Cloudflare API token with DNS write permissions
- An OpenShift [pull secret](https://console.redhat.com/openshift/install/pull-secret)
- An SSH key pair for cluster node access

## Getting started

Clone the repository and set up the Python virtual environment:

```shell
git clone https://github.com/exarep/iac.git
cd iac
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### Red Hat Demo Platform (RHDP)

If using RHDP, order the [AWS Blank Open Platform](https://catalog.demo.redhat.com/catalog/babylon-catalog-prod?item=babylon-catalog-prod/sandboxes-gpte.sandbox-open.prod&utm_source=webapp&utm_medium=share-link). Once the order completes, configure your local AWS environment with the credentials provided.

## Credentials

All playbooks read credentials from local files. Nothing is stored in the repository.

| Credential               | Source                                           | Variable                 |
| ---                      | ---                                              | ---                      |
| AWS access key           | `~/.aws/credentials`                             | `aws_profile`            |
| Cloudflare API token     | `~/.cloudflare/intexarep-exarep-dns-write`       | `cloudflare_token_file`  |
| OpenShift pull secret    | `~/.pull-secret.txt`                             | `pull_secret_file`       |
| SSH public key           | `~/.ssh/exarep.pub`                              | `cluster_public_key_file`|

### Configure AWS CLI

```shell
aws configure
AWS Access Key ID []: <your-access-key>
AWS Secret Access Key []: <your-secret-key>
Default region name [us-east-2]:
Default output format [json]:
```

To use a non-default AWS profile, pass `-e aws_profile=myprofile` when running any playbook.

## Project structure

```
iac/
├── ansible.cfg
├── requirements.txt
├── inventory/
│   ├── hosts.yaml
│   └── group_vars/
│       └── all.yaml
└── playbooks/
    ├── templates/
    │   └── install-config.yaml.j2
    ├── pb-setup-dns.yaml
    ├── pb-setup-secrets.yaml
    └── pb-create-hub-cluster.yaml
```

All variables are defined in `inventory/group_vars/all.yaml`. Playbook-specific templates live in `playbooks/templates/`.

## Inventory variables

| Variable                  | Description                                   | Default                |
| ---                       | ---                                           | ---                    |
| `aws_region`              | AWS region for all resources                  | `us-east-2`            |
| `base_domain`             | Root domain                                   | `exarep.com`           |
| `cloudflare_token_file`   | Path to Cloudflare API token file             | `~/.cloudflare/intexarep-exarep-dns-write` |
| `cluster_public_key_file` | Path to SSH public key for cluster nodes      | `~/.ssh/exarep.pub`    |
| `pull_secret_file`        | Path to OpenShift pull secret                 | `~/.pull-secret.txt`   |
| `dns_zones`               | List of Route 53 hosted zones to create       | `cluster.exarep.com`   |
| `cloudflare_dns_records`  | List of DNS records to create in Cloudflare   | See below              |
| `hub_cluster`             | Hub cluster configuration                     | See below              |
| `secrets_manager`         | AWS Secrets Manager configuration             | See below              |

## Playbooks

| Playbook                             | Description                                                                        |
| ---                                  | ---                                                                                |
| [`pb-setup-dns.yaml`](https://github.com/exarep/iac/blob/main/playbooks/pb-setup-dns.yaml){:target="_blank"}                  | Delegates DNS zones from Cloudflare to Route 53 and creates Cloudflare DNS records |
| [`pb-setup-secrets.yaml`](https://github.com/exarep/iac/blob/main/playbooks/pb-setup-secrets.yaml){:target="_blank"}             | Creates AWS Secrets Manager secrets with a dedicated KMS key                       |
| [`pb-create-hub-cluster.yaml`](https://github.com/exarep/iac/blob/main/playbooks/pb-create-hub-cluster.yaml){:target="_blank"}         | Creates the OpenShift hub cluster on AWS                                           |

---

### [pb-setup-dns.yaml](https://github.com/exarep/iac/blob/main/playbooks/pb-setup-dns.yaml){:target="_blank"}

Sets up DNS for the entire Exarep platform. It creates Route 53 hosted zones for subdomains that AWS needs to manage, delegates those zones from Cloudflare via NS records, and creates additional A/CNAME records for shared services.

Stale NS records in Cloudflare are automatically cleaned up on each run, so if a Route 53 hosted zone is recreated with new nameservers, the old delegation records are removed before the correct ones are added.

```shell
ansible-playbook playbooks/pb-setup-dns.yaml
```

#### What it does

1. Validates that AWS and Cloudflare credentials are present
2. Looks up the Cloudflare zone ID for `exarep.com`
3. Creates Route 53 hosted zones for each entry in `dns_zones`
4. Retrieves the NS records assigned by Route 53
5. Fetches existing NS records from Cloudflare and deletes any that are stale
6. Creates NS delegation records in Cloudflare pointing to Route 53
7. Creates additional DNS records (A, CNAME) in Cloudflare

#### Delegated zones

| Zone                   | Purpose               |
| ---                    | ---                   |
| `cluster.exarep.com`   | OpenShift cluster DNS |

#### Cloudflare DNS records

| Type  | Name               | Target                |
| ---   | ---                | ---                   |
| A     | `dns.exarep.com`   | `1.1.1.1`             |
| CNAME | `ntp.exarep.com`   | `time.cloudflare.com` |
| CNAME | `docs.exarep.com`  | `exarep.github.io`    |

---

### [pb-setup-secrets.yaml](https://github.com/exarep/iac/blob/main/playbooks/pb-setup-secrets.yaml){:target="_blank"}

Creates a dedicated KMS encryption key and provisions secrets in AWS Secrets Manager for the Exarep project. Each secret is created with a placeholder value and should be populated after the playbook runs.

```shell
ansible-playbook playbooks/pb-setup-secrets.yaml
```

#### What it does

1. Validates that AWS credentials are present
2. Creates a KMS symmetric encryption key aliased `exarep-secrets`
3. Creates each secret defined in `secrets_manager.secrets`, encrypted with the KMS key
4. Displays a summary with the KMS key ARN and all secret names

#### Secrets

| Secret                       | Description                                              |
| ---                          | ---                                                      |
| `exarep/aws-credentials`     | AWS credentials for External Secrets Operator (JSON)     |
| `exarep/test`                | Test secret for validating External Secrets Operator      |
| `exarep/cloudflare-api-token` | Cloudflare API token for DNS management                 |
| `exarep/pull-secret`         | OpenShift pull secret for cluster provisioning           |
| `exarep/cluster-ssh-key`     | SSH private key for cluster node access                  |

After running the playbook, populate each secret:

```shell
aws secretsmanager put-secret-value \
  --secret-id exarep/<name> \
  --secret-string '<value>'
```

---

### [pb-create-hub-cluster.yaml](https://github.com/exarep/iac/blob/main/playbooks/pb-create-hub-cluster.yaml){:target="_blank"}

Creates the OpenShift hub cluster at `hub.cluster.exarep.com` using `openshift-install` IPI (Installer-Provisioned Infrastructure) on AWS.

The playbook is idempotent. If `metadata.json` already exists in the cluster directory, the install is skipped entirely.

```shell
ansible-playbook playbooks/pb-create-hub-cluster.yaml
```

#### What it does

1. Validates that AWS credentials, pull secret, and SSH public key are present
2. Verifies that `openshift-install` is available on the PATH
3. Creates a cluster working directory at `iac/clusters/hub/`
4. Renders `install-config.yaml` from the Jinja template at `playbooks/templates/install-config.yaml.j2`
5. Backs up the install config (since `openshift-install` consumes the original)
6. Runs `openshift-install create cluster`
7. Cleans up completed and failed pods across all namespaces
8. Displays the console URL, API endpoint, and paths to kubeconfig and kubeadmin credentials

#### Cluster specification

| Parameter          | Value            |
| ---                | ---              |
| Cluster name       | `hub`            |
| Base domain        | `cluster.exarep.com` |
| Region             | `us-east-2`      |
| Control plane      | 3x `m6i.2xlarge` |
| Compute            | 3x `m6i.2xlarge` |
| Network plugin     | OVNKubernetes    |
| Cluster network    | `10.128.0.0/14`  |
| Service network    | `172.30.0.0/16`  |
| Machine network    | `10.0.0.0/16`    |

#### Post-install access

After the cluster is created, the playbook outputs the key access details:

| Resource           | Location                                                              |
| ---                | ---                                                                   |
| Web console        | `https://console-openshift-console.apps.hub.cluster.exarep.com`       |
| API server         | `https://api.hub.cluster.exarep.com:6443`                             |
| Kubeconfig         | `iac/clusters/hub/auth/kubeconfig`                                    |
| Kubeadmin password | `iac/clusters/hub/auth/kubeadmin-password`                            |

```shell
export KUBECONFIG=clusters/hub/auth/kubeconfig
oc whoami
```

## Execution order

For a fresh environment, clone the repo and run the playbooks in order:

```shell
git clone https://github.com/exarep/iac.git
cd iac
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

ansible-playbook playbooks/pb-setup-dns.yaml
ansible-playbook playbooks/pb-setup-secrets.yaml
ansible-playbook playbooks/pb-create-hub-cluster.yaml
```

DNS delegation must be in place before the cluster installer can create the required Route 53 records for the API and ingress endpoints.
