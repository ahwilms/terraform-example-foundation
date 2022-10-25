# terraform-example-foundation

This example repo shows how the Terraform modules for the Cloud Foundation Toolkit (CFT) can be composed to build a secure Google Cloud foundation, following the [Google Cloud security foundations guide](https://cloud.google.com/architecture/security-foundations).
The supplied structure and code is intended to form a starting point for building your own foundation, with pragmatic defaults that you can customize to meet your own requirements. 
Currently, step 0 is manually executed. From step 1 onwards, the Terraform code is deployed by using either [Google Cloud Build](https://cloud.google.com/build) (by default) or Jenkins.
The supplied structure and code is intended to form a starting point for building your own foundation, with pragmatic defaults that you can customize to meet your own requirements. Currently, step 0 is manually executed.
Cloud Build was chosen as the default to allow teams to quickly get started without needing to deploy a CI/CD tool, although you can use your preferred tool.

## Overview

This repo contains several distinct Terraform projects, each within their own directory that you must apply separately, in sequence.
Each of these Terraform projects are to be layered on top of each other, running in the following order.

### [0. bootstrap](./0-bootstrap/)

This stage executes the [CFT Bootstrap module](https://github.com/terraform-google-modules/terraform-google-bootstrap) which bootstraps an existing Google Cloud organization, creating all the required Google Cloud resources and permissions to start using CFT.
For [CI/CD Pipelines](/docs/GLOSSARY.md#foundation-cicd-pipeline), you can use either Cloud Build (by default) or Jenkins. If you want to use Jenkins instead of Cloud Build, see [README-Jenkins](./0-bootstrap/README-Jenkins.md) on how to use the included Jenkins sub-module.

The bootstrap step includes:

- The `prj-b-seed` project, which contains:
  - Terraform state bucket
  - Custom service accounts used by Terraform to create new resources in Google Cloud
- The `prj-b-cicd` project, which contains:
  - A [CI/CD Pipeline](/docs/GLOSSARY.md#foundation-cicd-pipeline) implemented with either Cloud Build or Jenkins
  - If using Cloud Build:
    - Cloud Source Repository
    - Artifact Registry
  - If using Jenkins:
    - A Google Compute Engine instance configured as a Jenkins Agent
    - A custom service account to run the Compute Engine instances
    - VPN connection with on-prem (or wherever your Jenkins Controller is located)

It is a best practice to separate concerns by having two projects: one for the CFT resources and one for the CI/CD tool.
The `prj-b-seed` project stores the Terraform state and has the service accounts that can create and modify infrastructure.
On the other hand, the deployment of that infrastructure is coordinated by the CI/CD tool of your choice, and allocated in a second project named `prj-b-cicd`.

To further separate the concerns at the IAM level, a distinct service account is created for each stage.
If using Cloud Build, these service accounts are used directly in the pipeline to execute the pipeline steps (`plan` or `apply`).
In this configuration, the baseline permissions of the CI/CD tool are unchanged, and the Terraform custom service accounts are granted the IAM permissions required to build the foundation.

If using Jenkins, the CI/CD tool account (`sa-jenkins-agent-gce@prj-b-cicd-xxxx.iam.gserviceaccount.com`) is granted access to generate tokens over the Terraform custom service accounts.
In this configuration, the baseline permissions of the CI/CD tool are limited, and the Terraform custom service accounts are granted the IAM permissions required to build the foundation.

After executing this step, you will have the following structure:

```
example-organization/
└── fldr-bootstrap
    ├── prj-b-cicd
    └── prj-b-seed
```

When this step uses the Cloud Build submodule, it sets up Cloud Build and Cloud Source Repositories for each of the stages below.
Triggers are configured to run a `terraform plan` for any non-environment branch and `terraform apply` when changes are merged to an environment branch (`development`, `non-production` & `production`).
For usage instructions, see [0-bootstrap README](./0-bootstrap/README.md).

### [1. org](./1-org/)

The purpose of this stage is to set up the common folder used for projects that contain shared resources such as DNS hub, Cloud Interconnect, Security Command Center notification, organization-level secrets, network hub, and organization-level logging.
This step will create the following folders and projects:

```
example-organization
└── fldr-common
    ├── prj-c-logging
    ├── prj-c-base-net-hub
    ├── prj-c-billing-logs
    ├── prj-c-dns-hub
    ├── prj-c-interconnect
    ├── prj-c-restricted-net-hub
    ├── prj-c-scc
    └── prj-c-secrets
```

#### Logs

Among the eight projects created under the common folder, two projects (`prj-c-logging`, `prj-c-billing-logs`) are used for logging.
The first one for organization-wide audit logs, and the latter for billing logs.
In both cases the logs are collected into BigQuery datasets which can then be used for general querying, dashboarding, and reporting. Logs are also exported to Pub/Sub, a Cloud Storage bucket, and a log bucket.

**Notes**:

- The log export to the Cloud Storage bucket has optional object versioning support using `log_export_storage_versioning`.
- The various audit log types that are captured in BigQuery are retained for 30 days.
- For billing data, a BigQuery dataset is created with permissions attached. However, you will need to configure a [billing export manually](https://cloud.google.com/billing/docs/how-to/export-data-bigquery), as there is no easy way to automate this at the moment.

#### DNS hub

Another project created under the common folder. This project will host the DNS hub for the organization.

#### Interconnect

Another project created under the common folder. This project will host the [Dedicated Interconnect connection](https://cloud.google.com/network-connectivity/docs/interconnect/concepts/terminology#elements) for the organization. In case of the Partner Interconnect, this project is unused and the [VLAN attachments](https://cloud.google.com/network-connectivity/docs/interconnect/concepts/terminology#for-partner-interconnect) will be placed directly into the corresponding hub projects.

#### SCC notification

Another project created under the common folder. This project will host the Security Command Center notification resources at the organization level.
This project will contain a Pub/Sub topic and subscription, and a [Security Command Center notification](https://cloud.google.com/security-command-center/docs/how-to-notifications) that is configured to send all new findings to the topic created.
You can adjust the filter when deploying this step.

#### Secrets

Another project created under the common folder. This project is allocated for [Secret Manager](https://cloud.google.com/secret-manager) for secrets shared by the organization.

For usage instructions, see the [org step README](./1-org/README.md).

### [2. environments](./2-environments/)

The purpose of this stage is to set up the environments folders used for projects that contain monitoring, secrets, and networking projects.
This will create the following folder and project structure:

```
example-organization
└── fldr-development
    ├── prj-d-monitoring
    ├── prj-d-secrets
    ├── prj-d-shared-base
    └── prj-d-shared-restricted
└── fldr-non-production
    ├── prj-n-monitoring
    ├── prj-n-secrets
    ├── prj-n-shared-base
    └── prj-n-shared-restricted
└── fldr-production
    ├── prj-p-monitoring
    ├── prj-p-secrets
    ├── prj-p-shared-base
    └── prj-p-shared-restricted
```

#### Monitoring

Under the environment folder, a project is created for each environment (`development`, `non-production`, and `production`), which is intended to be used as the [Cloud Monitoring scoping project](https://cloud.google.com/monitoring/settings) for all projects in that environment.
Please note that creating the [workspace and linking projects](https://cloud.google.com/monitoring/workspaces/create) can currently only be completed through the Cloud Console.
If you have strong IAM requirements for these monitoring workspaces, it is worth considering creating these at a more granular level, such as per business unit or per application.

#### Networking

Under the environment folder, two projects, one for base and another for restricted network, are created for each environment (`development`, `non-production`, and `production`) which is intended to be used as a [Shared VPC Host project](https://cloud.google.com/vpc/docs/shared-vpc) for all projects in that environment.
This stage only creates the projects and enables the correct APIs, the following networks stages, [3-networks-dual-svpc](./3-networks-dual-svpc/) and [3-networks-hub-and-spoke](./3-networks-hub-and-spoke/), create the actual Shared VPC networks.

#### Secrets

Under the environment folder, a project is created for each environment (`development`, `non-production`, and `production`), which is intended to be used by [Secret Manager](https://cloud.google.com/secret-manager) for secrets shared by the environment.

For usage instructions, see the [environments step README](./2-environments/README.md).

### [3. networks-dual-svpc](./3-networks-dual-svpc/)

This step focuses on creating a [Shared VPC](https://cloud.google.com/architecture/security-foundations/networking#vpcsharedvpc-id7-1-shared-vpc-) for each environment (`development`, `non-production`, and `production`) in a standard configuration with a reasonable security baseline. Currently, this includes:

- (Optional) Example subnets for `development`, `non-production`, and `production` inclusive of secondary ranges for those that want to use GKE.
- Hierarchical firewall policy for remote access to [VMs through IAP](https://cloud.google.com/iap/docs/using-tcp-forwarding), without needing public IPs.
- Hierarchical firewall policy for [load-balancing health checks](https://cloud.google.com/load-balancing/docs/health-checks#firewall_rules).
- Hierarchical firewall policy for [Windows KMS activation](https://cloud.google.com/compute/docs/instances/windows/creating-managing-windows-instances#kms-server).
- [Private service networking](https://cloud.google.com/vpc/docs/configure-private-services-access) configured to enable workload dependant resources like Cloud SQL.
- Base Shared VPC with [private.googleapis.com](https://cloud.google.com/vpc/docs/configure-private-google-access#private-domains) configured for base access to googleapis.com and gcr.io. Route added for VIP so no internet access is required to access APIs.
- Restricted Shared VPC with [restricted.googleapis.com](https://cloud.google.com/vpc-service-controls/docs/supported-products) configured for restricted access to googleapis.com and gcr.io. Route added for VIP so no internet access is required to access APIs.
- Default routes to internet removed, with tag based route `egress-internet` required on VMs in order to reach the internet.
- (Optional) Cloud NAT configured for all subnets with logging and static outbound IPs.
- Default Cloud DNS policy applied, with DNS logging and [inbound query forwarding](https://cloud.google.com/dns/docs/overview#dns-forwarding-methods) turned on.

For usage instructions, see the [Shared VPC networks README](./3-networks-dual-svpc/README.md).

### [3. networks-hub-and-spoke](./3-networks-hub-and-spoke/)

This step configures the same network resources that the step 3-networks-dual-svpc does, but this time it makes usage of the architecture based on the [Hub and Spoke](https://cloud.google.com/architecture/security-foundations/networking#hub-and-spoke) reference network model.

For usage instructions, see the [hub-and-spoke networks README](./3-networks-hub-and-spoke/README.md).

### [4. projects](./4-projects/)

This step is focused on creating service projects with a standard configuration that are attached to the Shared VPC that was created in the previous step and the application infrastructure pipelines.
Running this code as-is should generate a structure as shown below:

```
example-organization/
└── fldr-development
    ├── prj-bu1-d-env-secrets
    ├── prj-bu1-d-sample-floating
    ├── prj-bu1-d-sample-base
    ├── prj-bu1-d-sample-restrict
    ├── prj-bu1-d-sample-peering
    ├── prj-bu2-d-env-secrets
    ├── prj-bu2-d-sample-floating
    ├── prj-bu2-d-sample-base
    ├── prj-bu2-d-sample-restrict
    └── prj-bu2-d-sample-peering
└── fldr-non-production
    ├── prj-bu1-n-env-secrets
    ├── prj-bu1-n-sample-floating
    ├── prj-bu1-n-sample-base
    ├── prj-bu1-n-sample-restrict
    ├── prj-bu1-n-sample-peering
    ├── prj-bu2-n-env-secrets
    ├── prj-bu2-n-sample-floating
    ├── prj-bu2-n-sample-base
    ├── prj-bu2-n-sample-restrict
    └── prj-bu2-n-sample-peering
└── fldr-production
    ├── prj-bu1-p-env-secrets
    ├── prj-bu1-p-sample-floating
    ├── prj-bu1-p-sample-base
    ├── prj-bu1-p-sample-restrict
    ├── prj-bu1-p-sample-peering
    ├── prj-bu2-p-env-secrets
    ├── prj-bu2-p-sample-floating
    ├── prj-bu2-p-sample-base
    ├── prj-bu2-p-sample-restrict
    └── prj-bu2-p-sample-peering
└── fldr-common
    ├── prj-bu1-c-infra-pipeline
    └── prj-bu2-c-infra-pipeline
```

The code in this step includes two options for creating projects.
The first is the standard projects module which creates a project per environment, and the second creates a standalone project for one environment.
If relevant for your use case, there are also two optional submodules which can be used to create a subnet per project, and a dedicated private DNS zone per project.

For usage instructions, see the [projects step README](./4-projects/README.md).

### [5. app-infra](./5-app-infra/)

The purpose of this step is to deploy a simple [Compute Engine](https://cloud.google.com/compute/) instance in one of the business unit projects using the infra pipeline set up in 4-projects.

For usage instructions, see the [app-infra step README](./5-app-infra/README.md).

### Final View

After all steps have been executed, your Google Cloud organization should have the following structure, with projects being the lowest nodes in the tree:

```
example-organization
└── fldr-common
    ├── prj-c-logging
    ├── prj-c-base-net-hub
    ├── prj-c-billing-logs
    ├── prj-c-dns-hub
    ├── prj-c-interconnect
    ├── prj-c-restricted-net-hub
    ├── prj-c-scc
    ├── prj-c-secrets
    ├── prj-bu1-c-infra-pipeline
    └── prj-bu2-c-infra-pipeline
└── fldr-development
    ├── prj-bu1-d-env-secrets
    ├── prj-bu1-d-sample-floating
    ├── prj-bu1-d-sample-base
    ├── prj-bu1-d-sample-restrict
    ├── prj-bu1-d-sample-peering
    ├── prj-bu2-d-env-secrets
    ├── prj-bu2-d-sample-floating
    ├── prj-bu2-d-sample-base
    ├── prj-bu2-d-sample-restrict
    ├── prj-bu2-d-sample-peering
    ├── prj-d-monitoring
    ├── prj-d-secrets
    ├── prj-d-shared-base
    └── prj-d-shared-restricted
└── fldr-non-production
    ├── prj-bu1-n-env-secrets
    ├── prj-bu1-n-sample-floating
    ├── prj-bu1-n-sample-base
    ├── prj-bu1-n-sample-restrict
    ├── prj-bu1-n-sample-peering
    ├── prj-bu2-n-env-secrets
    ├── prj-bu2-n-sample-floating
    ├── prj-bu2-n-sample-base
    ├── prj-bu2-n-sample-restrict
    ├── prj-bu2-n-sample-peering
    ├── prj-n-monitoring
    ├── prj-n-secrets
    ├── prj-n-shared-base
    └── prj-n-shared-restricted
└── fldr-production
    ├── prj-bu1-p-env-secrets
    ├── prj-bu1-p-sample-floating
    ├── prj-bu1-p-sample-base
    ├── prj-bu1-p-sample-restrict
    ├── prj-bu1-p-sample-peering
    ├── prj-bu2-p-env-secrets
    ├── prj-bu2-p-sample-floating
    ├── prj-bu2-p-sample-base
    ├── prj-bu2-p-sample-restrict
    ├── prj-bu2-p-sample-peering
    ├── prj-p-monitoring
    ├── prj-p-secrets
    ├── prj-p-shared-base
    └── prj-p-shared-restricted
└── fldr-bootstrap
    ├── prj-b-cicd
    └── prj-b-seed
```

### Branching strategy

There are three main branches, `development`, `non-production`, and `production`, that reflect the corresponding environments. These branches should be [protected](https://docs.github.com/en/github/administering-a-repository/about-protected-branches). When the [CI/CD Pipeline](/docs/GLOSSARY.md#foundation-cicd-pipeline) (whether Jenkins or Cloud Build) runs on a particular named branch (say for instance `development`), only the corresponding environment (`development`) is applied. An exception is the `shared` environment which is only applied when triggered on the `production` branch. This is because any changes in the `shared` environment may affect resources in other environments and can have adverse effects if not validated correctly.

Development happens on feature/bugfix branches (which can be named `feature/new-foo`, `bugfix/fix-bar`, etc.) and when complete, a [pull request (PR)](https://docs.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-requests) or [merge request (MR)](https://docs.gitlab.com/ee/user/project/merge_requests/) can be opened targeting the `development` branch. This will trigger the [CI/CD Pipeline](/docs/GLOSSARY.md#foundation-cicd-pipeline) to perform a plan and validate against all environments (`development`, `non-production`, `shared`, and `production`). After code review is complete and changes are validated, this branch can be merged into `development`. This will trigger a [CI/CD Pipeline](/docs/GLOSSARY.md#foundation-cicd-pipeline) that applies the latest changes in the `development` branch on the `development` environment.

Once validated in `development`, changes can be promoted to `non-production` by opening a PR/MR targeting the `non-production` branch and merging them. Similarly, changes can be promoted from `non-production` to `production`.

### Terraform-validator

This repo uses the [terraform-tools](https://cloud.google.com/docs/terraform/policy-validation/validate-policies) component of the `gcloud` CLI to validate the Terraform plans against a [library of Google Cloud policies](https://github.com/GoogleCloudPlatform/policy-library).

The [Scorecard bundle](https://github.com/GoogleCloudPlatform/policy-library/blob/master/docs/bundles/scorecard-v1.md) was used to create the [policy-library folder](./policy-library) with [one extra constraint](https://github.com/GoogleCloudPlatform/policy-library/blob/master/samples/serviceusage_allow_basic_apis.yaml) added.

See the [policy-library documentation](https://github.com/GoogleCloudPlatform/policy-library/blob/master/docs/index.md) if you need to add more constraints from the [samples folder](https://github.com/GoogleCloudPlatform/policy-library/tree/master/samples) in your configuration based in your type of workload.

Step 1-org has [instructions](./1-org/README.md#deploying-with-cloud-build) on the creation of the shared repository to host these policies.

### Optional variables

Some variables used to deploy the steps have default values. Check those **before deployment** to ensure they match your requirements. For more information, there are tables of inputs and outputs for the Terraform modules, each with a detailed description of their variables. Look for variables marked as **not required** in the section **Inputs** of these READMEs:

- Step 0-bootstrap: If you are using Cloud Build in the [CI/CD Pipeline](/docs/GLOSSARY.md#foundation-cicd-pipeline), check the main [README](./0-bootstrap/README.md#Inputs) of the step. If you are using Jenkins, check the [README](./0-bootstrap/modules/jenkins-agent/README.md#Inputs) of the module `jenkins-agent`.
- Step 1-org: The [README](./1-org/envs/shared/README.md#Inputs) of the environment `shared`.
- Step 2-environments: The READMEs of the environments [development](./2-environments/envs/development/README.md#Inputs), [non-production](./2-environments/envs/non-production/README.md#Inputs), and [production](./2-environments/envs/production/README.md#Inputs)
- Step 3-networks-dual-svpc: The READMEs of the environments [shared](./3-networks-dual-svpc/envs/shared/README.md#inputs), [development](./3-networks-dual-svpc/envs/development/README.md#Inputs), [non-production](./3-networks/envs/non-production/README.md#Inputs), and [production](./3-networks/envs/production/README.md#Inputs)
- Step 3-networks-hub-and-spoke: The READMEs of the environments [shared](./3-networks-hub-and-spoke/envs/shared/README.md#inputs), [development](./3-networks-hub-and-spoke/envs/development/README.md#Inputs), [non-production](./3-networks/envs/non-production/README.md#Inputs) and [production](./3-networks/envs/production/README.md#Inputs)
- Step 4-projects: The READMEs of the environments [shared](./4-projects/business_unit_1/shared/README.md#inputs), [development](./4-projects/business_unit_1/development/README.md#Inputs), [non-production](./4-projects/business_unit_1/non-production/README.md#Inputs) and [production](./4-projects/business_unit_1/production/README.md#Inputs)

## Errata summary

Refer to the [errata summary](./ERRATA.md) for an overview of the delta between the example foundation repository and the [Google Cloud security foundations guide](https://cloud.google.com/architecture/security-foundations).

## Contributing

Refer to the [contribution guidelines](./CONTRIBUTING.md) for information on contributing to this module.
