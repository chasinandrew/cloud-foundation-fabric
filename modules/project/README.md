# Project Module

This module implements the creation and management of one GCP project including IAM, organization policies, Shared VPC host or service attachment, service API activation, and tag attachment. It also offers a convenient way to refer to managed service identities (aka robot service accounts) for APIs.

## TOC

<!-- BEGIN TOC -->
- [TOC](#toc)
- [Basic Project Creation](#basic-project-creation)
- [IAM](#iam)
  - [Authoritative IAM](#authoritative-iam)
  - [Additive IAM](#additive-iam)
  - [Additive IAM by Member](#additive-iam-by-member)
  - [Service Identities and Authoritative IAM](#service-identities-and-authoritative-iam)
  - [Using Shortcodes for Service Identities in Additive Iam](#using-shortcodes-for-service-identities-in-additive-iam)
  - [Service Identities Requiring Manual Iam Grants](#service-identities-requiring-manual-iam-grants)
- [Shared VPC](#shared-vpc)
- [Organization Policies](#organization-policies)
  - [Organization Policy Factory](#organization-policy-factory)
- [Log Sinks](#log-sinks)
- [Data Access Logs](#data-access-logs)
- [Cloud Kms Encryption Keys](#cloud-kms-encryption-keys)
- [Tags](#tags)
- [Outputs](#outputs)
- [Files](#files)
- [Variables](#variables)
- [Outputs](#outputs)
<!-- END TOC -->

## Basic Project Creation

```hcl
module "project" {
  source          = "./fabric/modules/project"
  billing_account = "123456-123456-123456"
  name            = "myproject"
  parent          = "folders/1234567890"
  prefix          = "foo"
  services = [
    "container.googleapis.com",
    "stackdriver.googleapis.com"
  ]
}
# tftest modules=1 resources=3 inventory=basic.yaml
```

## IAM

IAM is managed via several variables that implement different levels of control:

- `group_iam` and `iam` configure authoritative bindings that manage individual roles exclusively, mapping to the [`google_project_iam_binding`](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_project_iam#google_project_iam_binding) resource
- `iam_additive` and `iam_additive_members` configure additive bindings that only manage individual role/member pairs, mapping to the [`google_project_iam_member`](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_project_iam#google_project_iam_member) resource
- `iam_policy` which controls the entire IAM policy for the project, where any binding created outside this module (eg in the console) will be removed at each `terraform apply` cycle regardless of the role

The authoritative and additive approaches can be used together, provided different roles are managed by each. The IAM policy is incompatible with the other approaches, and must be used with extreme care.

Be mindful about service identity roles when using authoritative IAM, as you might inadvertently remove a role from a [service identity](https://cloud.google.com/iam/docs/service-account-types#google-managed) or default service account. For example, using `roles/editor` with `iam` or `group_iam` will remove the default permissions for the Cloud Services identity. A simple workaround for these scenarios is described below.

### Authoritative IAM

The `iam` variable is based on role keys and is typically used for service accounts, or where member values can be dynamic and would create potential problems in the underlying `for_each` cycle.

```hcl
locals {
  gke_service_account = "my_gke_service_account"
}

module "project" {
  source          = "./fabric/modules/project"
  billing_account = "123456-123456-123456"
  name            = "project-example"
  parent          = "folders/1234567890"
  prefix          = "foo"
  services = [
    "container.googleapis.com",
    "stackdriver.googleapis.com"
  ]
  iam = {
    "roles/container.hostServiceAgentUser" = [
      "serviceAccount:${local.gke_service_account}"
    ]
  }
}
# tftest modules=1 resources=4 inventory=iam-authoritative.yaml
```

The `group_iam` variable uses group email addresses as keys and is a convenient way to assign roles to humans following Google's best practices. The end result is readable code that also serves as documentation.

```hcl
module "project" {
  source          = "./fabric/modules/project"
  billing_account = "123456-123456-123456"
  name            = "project-example"
  parent          = "folders/1234567890"
  prefix          = "foo"
  group_iam = {
    "gcp-security-admins@example.com" = [
      "roles/cloudasset.owner",
      "roles/cloudsupport.techSupportEditor",
      "roles/iam.securityReviewer",
      "roles/logging.admin",
    ]
  }
}
# tftest modules=1 resources=5 inventory=iam-group.yaml
```

### Additive IAM

Additive IAM is typically used where bindings for specific roles are controlled by different modules or in different Terraform stages. One example is when the project is created by one team but a different team manages service account creation for the project, and some of the project-level roles overlap in the two configurations.

```hcl
module "project" {
  source = "./fabric/modules/project"
  name   = "project-example"
  iam_additive = {
    "roles/viewer" = [
      "group:one@example.org",
      "group:two@xample.org"
    ],
    "roles/storage.objectAdmin" = [
      "group:two@example.org"
    ],
    "roles/owner" = [
      "group:three@example.org"
    ],
  }
}
# tftest modules=1 resources=5 inventory=iam-additive.yaml
```

### Additive IAM by Member

```hcl
module "project" {
  source = "./fabric/modules/project"
  name   = "project-example"
  iam_additive_members = {
    "user:one@example.org" = ["roles/owner"]
    "user:two@example.org" = ["roles/owner", "roles/editor"]
  }

}
# tftest modules=1 resources=4 inventory=iam-additive-members.yaml
```

### Service Identities and Authoritative IAM

As mentioned above, there are cases where authoritative management of specific IAM roles results in removal of default bindings from service identities. One example is outlined below, with a simple workaround leveraging the `service_accounts` output to identify the service identity. A full list of service identities and their roles can be found [here](https://cloud.google.com/iam/docs/service-agents).

```hcl
module "project" {
  source = "./fabric/modules/project"
  name   = "project-example"
  group_iam = {
    "foo@example.com" = [
      "roles/editor"
    ]
  }
  iam = {
    "roles/editor" = [
      "serviceAccount:${module.project.service_accounts.cloud_services}"
    ]
  }
}
# tftest modules=1 resources=2
```

### Using Shortcodes for Service Identities in Additive Iam

Most Service Identities contains project number in their e-mail address and this prevents additive IAM to work, as these values are not known at moment of execution of `terraform plan` (its not an issue for authoritative IAM). To refer current project Service Identities you may use shortcodes for Service Identities similarly as for `service_identity_iam` when configuring Shared VPC.

```hcl
module "project" {
  source = "./fabric/modules/project"
  name   = "project-example"

  services = [
    "run.googleapis.com",
    "container.googleapis.com",
  ]

  iam_additive = {
    "roles/editor"                         = ["cloudservices"]
    "roles/vpcaccess.user"                 = ["cloudrun"]
    "roles/container.hostServiceAgentUser" = ["container-engine"]
  }
}
# tftest modules=1 resources=6
```

### Service Identities Requiring Manual Iam Grants

The module will create service identities at project creation instead of creating of them at the time of first use. This allows granting these service identities roles in other projects, something which is usually necessary in a Shared VPC context.  

You can grant roles to service identities using the following construct:

```hcl
module "project" {
  source = "./fabric/modules/project"
  name   = "project-example"
  iam = {
    "roles/apigee.serviceAgent" = [
      "serviceAccount:${module.project.service_accounts.robots.apigee}"
    ]
  }
}
# tftest modules=1 resources=2
```

This table lists all affected services and roles that you need to grant to service identities

| service | service identity | role |
|---|---|---|
| apigee.googleapis.com | apigee | roles/apigee.serviceAgent |
| artifactregistry.googleapis.com | artifactregistry | roles/artifactregistry.serviceAgent |
| cloudasset.googleapis.com | cloudasset | roles/cloudasset.serviceAgent |
| cloudbuild.googleapis.com | cloudbuild | roles/cloudbuild.builds.builder |
| dataplex.googleapis.com | dataplex | roles/dataplex.serviceAgent |
| gkehub.googleapis.com | fleet | roles/gkehub.serviceAgent |
| meshconfig.googleapis.com | servicemesh | roles/anthosservicemesh.serviceAgent |
| multiclusteringress.googleapis.com | multicluster-ingress | roles/multiclusteringress.serviceAgent |
| pubsub.googleapis.com | pubsub | roles/pubsub.serviceAgent |
| sqladmin.googleapis.com | sqladmin | roles/cloudsql.serviceAgent |

## Shared VPC

The module allows managing Shared VPC status for both hosts and service projects, and includes a simple way of assigning Shared VPC roles to service identities.

You can enable Shared VPC Host at the project level and manage project service association independently.

```hcl
module "host-project" {
  source = "./fabric/modules/project"
  name   = "my-host-project"
  shared_vpc_host_config = {
    enabled = true
  }
}

module "service-project" {
  source = "./fabric/modules/project"
  name   = "my-service-project"
  shared_vpc_service_config = {
    host_project = module.host-project.project_id
    service_identity_iam = {
      "roles/compute.networkUser" = [
        "cloudservices", "container-engine"
      ]
      "roles/vpcaccess.user" = [
        "cloudrun"
      ]
      "roles/container.hostServiceAgentUser" = [
        "container-engine"
      ]
    }
  }
}
# tftest modules=2 resources=8 inventory=shared-vpc.yaml
```

The module allows also granting necessary permissions in host project to service identities by specifying which services will be used in service project in `grant_iam_for_services`.
```hcl
module "host-project" {
  source = "./fabric/modules/project"
  name   = "my-host-project"
  shared_vpc_host_config = {
    enabled = true
  }
}

module "service-project" {
  source = "./fabric/modules/project"
  name   = "my-service-project"
  services = [
    "container.googleapis.com",
  ]
  shared_vpc_service_config = {
    host_project       = module.host-project.project_id
    service_iam_grants = module.service-project.services
  }
}
# tftest modules=2 resources=9 inventory=shared-vpc-auto-grants.yaml
```

## Organization Policies

To manage organization policies, the `orgpolicy.googleapis.com` service should be enabled in the quota project.

```hcl
module "project" {
  source          = "./fabric/modules/project"
  billing_account = "123456-123456-123456"
  name            = "project-example"
  parent          = "folders/1234567890"
  prefix          = "foo"
  org_policies = {
    "compute.disableGuestAttributesAccess" = {
      rules = [{ enforce = true }]
    }
    "compute.skipDefaultNetworkCreation" = {
      rules = [{ enforce = true }]
    }
    "iam.disableServiceAccountKeyCreation" = {
      rules = [{ enforce = true }]
    }
    "iam.disableServiceAccountKeyUpload" = {
      rules = [
        {
          condition = {
            expression  = "resource.matchTagId('tagKeys/1234', 'tagValues/1234')"
            title       = "condition"
            description = "test condition"
            location    = "somewhere"
          }
          enforce = true
        },
        {
          enforce = false
        }
      ]
    }
    "iam.allowedPolicyMemberDomains" = {
      rules = [{
        allow = {
          values = ["C0xxxxxxx", "C0yyyyyyy"]
        }
      }]
    }
    "compute.trustedImageProjects" = {
      rules = [{
        allow = {
          values = ["projects/my-project"]
        }
      }]
    }
    "compute.vmExternalIpAccess" = {
      rules = [{ deny = { all = true } }]
    }
  }
}
# tftest modules=1 resources=8 inventory=org-policies.yaml
```

### Organization Policy Factory

Organization policies can be loaded from a directory containing YAML files where each file defines one or more constraints. The structure of the YAML files is exactly the same as the `org_policies` variable.

Note that constraints defined via `org_policies` take precedence over those in `org_policies_data_path`. In other words, if you specify the same constraint in a YAML file *and* in the `org_policies` variable, the latter will take priority.

The example below deploys a few organization policies split between two YAML files.

```hcl
module "project" {
  source                 = "./fabric/modules/project"
  billing_account        = "123456-123456-123456"
  name                   = "project-example"
  parent                 = "folders/1234567890"
  prefix                 = "foo"
  org_policies_data_path = "configs/org-policies/"
}
# tftest modules=1 resources=8 files=boolean,list inventory=org-policies.yaml
```

```yaml
# tftest-file id=boolean path=configs/org-policies/boolean.yaml
compute.disableGuestAttributesAccess:
  rules:
  - enforce: true
compute.skipDefaultNetworkCreation:
  rules:
  - enforce: true
iam.disableServiceAccountKeyCreation:
  rules:
  - enforce: true
iam.disableServiceAccountKeyUpload:
  rules:
  - condition:
      description: test condition
      expression: resource.matchTagId('tagKeys/1234', 'tagValues/1234')
      location: somewhere
      title: condition
    enforce: true
  - enforce: false
```

```yaml
# tftest-file id=list path=configs/org-policies/list.yaml
compute.trustedImageProjects:
  rules:
  - allow:
      values:
      - projects/my-project
compute.vmExternalIpAccess:
  rules:
  - deny:
      all: true
iam.allowedPolicyMemberDomains:
  rules:
  - allow:
      values:
      - C0xxxxxxx
      - C0yyyyyyy
```

## Log Sinks

```hcl
module "gcs" {
  source        = "./fabric/modules/gcs"
  project_id    = var.project_id
  name          = "gcs_sink"
  force_destroy = true
}

module "dataset" {
  source     = "./fabric/modules/bigquery-dataset"
  project_id = var.project_id
  id         = "bq_sink"
}

module "pubsub" {
  source     = "./fabric/modules/pubsub"
  project_id = var.project_id
  name       = "pubsub_sink"
}

module "bucket" {
  source      = "./fabric/modules/logging-bucket"
  parent_type = "project"
  parent      = "my-project"
  id          = "bucket"
}

module "project-host" {
  source          = "./fabric/modules/project"
  name            = "my-project"
  billing_account = "123456-123456-123456"
  parent          = "folders/1234567890"
  logging_sinks = {
    warnings = {
      destination = module.gcs.id
      filter      = "severity=WARNING"
      type        = "storage"
    }
    info = {
      destination = module.dataset.id
      filter      = "severity=INFO"
      type        = "bigquery"
    }
    notice = {
      destination = module.pubsub.id
      filter      = "severity=NOTICE"
      type        = "pubsub"
    }
    debug = {
      destination = module.bucket.id
      filter      = "severity=DEBUG"
      exclusions = {
        no-compute = "logName:compute"
      }
      type = "logging"
    }
  }
  logging_exclusions = {
    no-gce-instances = "resource.type=gce_instance"
  }
}
# tftest modules=5 resources=14 inventory=logging.yaml
```

## Data Access Logs

Activation of data access logs can be controlled via the `logging_data_access` variable. If the `iam_bindings_authoritative` variable is used to set a resource-level IAM policy, the data access log configuration will also be authoritative as part of the policy.

This example shows how to set a non-authoritative access log configuration:

```hcl
module "project" {
  source          = "./fabric/modules/project"
  name            = "my-project"
  billing_account = "123456-123456-123456"
  parent          = "folders/1234567890"
  logging_data_access = {
    allServices = {
      # logs for principals listed here will be excluded
      ADMIN_READ = ["group:organization-admins@example.org"]
    }
    "storage.googleapis.com" = {
      DATA_READ  = []
      DATA_WRITE = []
    }
  }
}
# tftest modules=1 resources=3 inventory=logging-data-access.yaml
```

While this sets an authoritative policies that has exclusive control of both IAM bindings for all roles and data access log configuration, and should be used with extreme care:

```hcl
module "project" {
  source          = "./fabric/modules/project"
  name            = "my-project"
  billing_account = "123456-123456-123456"
  parent          = "folders/1234567890"
  iam_policy = {
    "roles/owner"                             = ["group:org-admins@example.com"]
    "roles/resourcemanager.folderAdmin"       = ["group:org-admins@example.com"]
    "roles/resourcemanager.organizationAdmin" = ["group:org-admins@example.com"]
    "roles/resourcemanager.projectCreator"    = ["group:org-admins@example.com"]
  }
  logging_data_access = {
    allServices = {
      ADMIN_READ = ["group:organization-admins@example.org"]
    }
    "storage.googleapis.com" = {
      DATA_READ  = []
      DATA_WRITE = []
    }
  }
}
# tftest modules=1 resources=2 inventory=iam-policy.yaml
```

## Cloud Kms Encryption Keys

The module offers a simple, centralized way to assign `roles/cloudkms.cryptoKeyEncrypterDecrypter` to service identities.

```hcl
module "project" {
  source = "./fabric/modules/project"
  name   = "my-project"
  prefix = "foo"
  services = [
    "compute.googleapis.com",
    "storage.googleapis.com"
  ]
  service_encryption_key_ids = {
    compute = [
      "projects/kms-central-prj/locations/europe-west3/keyRings/my-keyring/cryptoKeys/europe3-gce",
      "projects/kms-central-prj/locations/europe-west4/keyRings/my-keyring/cryptoKeys/europe4-gce"
    ]
    storage = [
      "projects/kms-central-prj/locations/europe/keyRings/my-keyring/cryptoKeys/europe-gcs"
    ]
  }
}
# tftest modules=1 resources=7
```

## Tags

Refer to the [Creating and managing tags](https://cloud.google.com/resource-manager/docs/tags/tags-creating-and-managing) documentation for details on usage.

```hcl
module "org" {
  source          = "./fabric/modules/organization"
  organization_id = var.organization_id
  tags = {
    environment = {
      description = "Environment specification."
      iam         = null
      values = {
        dev  = null
        prod = null
      }
    }
  }
}

module "project" {
  source = "./fabric/modules/project"
  name   = "test-project"
  tag_bindings = {
    env-prod = module.org.tag_values["environment/prod"].id
    foo      = "tagValues/12345678"
  }
}
# tftest modules=2 resources=6
```

## Outputs

Most of this module's outputs depend on its resources, to allow Terraform to compute all dependencies required for the project to be correctly configured. This allows you to reference outputs like `project_id` in other modules or resources without having to worry about setting `depends_on` blocks manually.

One non-obvious output is `service_accounts`, which offers a simple way to discover service identities and default service accounts, and guarantees that service identities that require an API call to trigger creation (like GCS or BigQuery) exist before use.

```hcl
module "project" {
  source = "./fabric/modules/project"
  name   = "project-example"
  services = [
    "compute.googleapis.com"
  ]
}

output "compute_robot" {
  value = module.project.service_accounts.robots.compute
}
# tftest modules=1 resources=2 inventory:outputs.yaml
```

<!-- TFDOC OPTS files:1 -->
<!-- BEGIN TFDOC -->
## Files

| name | description | resources |
|---|---|---|
| [iam.tf](./iam.tf) | Generic and OSLogin-specific IAM bindings and roles. | <code>google_project_iam_binding</code> · <code>google_project_iam_custom_role</code> · <code>google_project_iam_member</code> · <code>google_project_iam_policy</code> |
| [logging.tf](./logging.tf) | Log sinks and supporting resources. | <code>google_bigquery_dataset_iam_member</code> · <code>google_logging_project_exclusion</code> · <code>google_logging_project_sink</code> · <code>google_project_iam_audit_config</code> · <code>google_project_iam_member</code> · <code>google_pubsub_topic_iam_member</code> · <code>google_storage_bucket_iam_member</code> |
| [main.tf](./main.tf) | Module-level locals and resources. | <code>google_compute_project_metadata_item</code> · <code>google_essential_contacts_contact</code> · <code>google_monitoring_monitored_project</code> · <code>google_project</code> · <code>google_project_service</code> · <code>google_resource_manager_lien</code> |
| [organization-policies.tf](./organization-policies.tf) | Project-level organization policies. | <code>google_org_policy_policy</code> |
| [outputs.tf](./outputs.tf) | Module outputs. |  |
| [service-accounts.tf](./service-accounts.tf) | Service identities and supporting resources. | <code>google_kms_crypto_key_iam_member</code> · <code>google_project_default_service_accounts</code> · <code>google_project_iam_member</code> · <code>google_project_service_identity</code> |
| [shared-vpc.tf](./shared-vpc.tf) | Shared VPC project-level configuration. | <code>google_compute_shared_vpc_host_project</code> · <code>google_compute_shared_vpc_service_project</code> · <code>google_project_iam_member</code> |
| [tags.tf](./tags.tf) | None | <code>google_tags_tag_binding</code> |
| [variables.tf](./variables.tf) | Module variables. |  |
| [versions.tf](./versions.tf) | Version pins. |  |
| [vpc-sc.tf](./vpc-sc.tf) | VPC-SC project-level perimeter configuration. | <code>google_access_context_manager_service_perimeter_resource</code> |

## Variables

| name | description | type | required | default |
|---|---|:---:|:---:|:---:|
| [name](variables.tf#L161) | Project name and id suffix. | <code>string</code> | ✓ |  |
| [auto_create_network](variables.tf#L17) | Whether to create the default network for the project. | <code>bool</code> |  | <code>false</code> |
| [billing_account](variables.tf#L23) | Billing account id. | <code>string</code> |  | <code>null</code> |
| [contacts](variables.tf#L29) | List of essential contacts for this resource. Must be in the form EMAIL -> [NOTIFICATION_TYPES]. Valid notification types are ALL, SUSPENSION, SECURITY, TECHNICAL, BILLING, LEGAL, PRODUCT_UPDATES. | <code>map&#40;list&#40;string&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [custom_roles](variables.tf#L36) | Map of role name => list of permissions to create in this project. | <code>map&#40;list&#40;string&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [default_service_account](variables.tf#L43) | Project default service account setting: can be one of `delete`, `deprivilege`, `disable`, or `keep`. | <code>string</code> |  | <code>&#34;keep&#34;</code> |
| [descriptive_name](variables.tf#L49) | Name of the project name. Used for project name instead of `name` variable. | <code>string</code> |  | <code>null</code> |
| [group_iam](variables.tf#L55) | Authoritative IAM binding for organization groups, in {GROUP_EMAIL => [ROLES]} format. Group emails need to be static. Can be used in combination with the `iam` variable. | <code>map&#40;list&#40;string&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [iam](variables.tf#L62) | IAM bindings in {ROLE => [MEMBERS]} format. | <code>map&#40;list&#40;string&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [iam_additive](variables.tf#L69) | IAM additive bindings in {ROLE => [MEMBERS]} format. | <code>map&#40;list&#40;string&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [iam_additive_members](variables.tf#L76) | IAM additive bindings in {MEMBERS => [ROLE]} format. This might break if members are dynamic values. | <code>map&#40;list&#40;string&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [iam_policy](variables.tf#L82) | IAM authoritative policy in {ROLE => [MEMBERS]} format. Roles and members not explicitly listed will be cleared, use with extreme caution. | <code>map&#40;list&#40;string&#41;&#41;</code> |  | <code>null</code> |
| [labels](variables.tf#L88) | Resource labels. | <code>map&#40;string&#41;</code> |  | <code>&#123;&#125;</code> |
| [lien_reason](variables.tf#L95) | If non-empty, creates a project lien with this description. | <code>string</code> |  | <code>&#34;&#34;</code> |
| [logging_data_access](variables.tf#L101) | Control activation of data access logs. Format is service => { log type => [exempted members]}. The special 'allServices' key denotes configuration for all services. | <code>map&#40;map&#40;list&#40;string&#41;&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [logging_exclusions](variables.tf#L116) | Logging exclusions for this project in the form {NAME -> FILTER}. | <code>map&#40;string&#41;</code> |  | <code>&#123;&#125;</code> |
| [logging_sinks](variables.tf#L123) | Logging sinks to create for this project. | <code title="map&#40;object&#40;&#123;&#10;  bq_partitioned_table &#61; optional&#40;bool&#41;&#10;  description          &#61; optional&#40;string&#41;&#10;  destination          &#61; string&#10;  disabled             &#61; optional&#40;bool, false&#41;&#10;  exclusions           &#61; optional&#40;map&#40;string&#41;, &#123;&#125;&#41;&#10;  filter               &#61; string&#10;  iam                  &#61; optional&#40;bool, true&#41;&#10;  type                 &#61; string&#10;  unique_writer        &#61; optional&#40;bool&#41;&#10;&#125;&#41;&#41;">map&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [metric_scopes](variables.tf#L154) | List of projects that will act as metric scopes for this project. | <code>list&#40;string&#41;</code> |  | <code>&#91;&#93;</code> |
| [org_policies](variables.tf#L166) | Organization policies applied to this project keyed by policy name. | <code title="map&#40;object&#40;&#123;&#10;  inherit_from_parent &#61; optional&#40;bool&#41; &#35; for list policies only.&#10;  reset               &#61; optional&#40;bool&#41;&#10;  rules &#61; optional&#40;list&#40;object&#40;&#123;&#10;    allow &#61; optional&#40;object&#40;&#123;&#10;      all    &#61; optional&#40;bool&#41;&#10;      values &#61; optional&#40;list&#40;string&#41;&#41;&#10;    &#125;&#41;&#41;&#10;    deny &#61; optional&#40;object&#40;&#123;&#10;      all    &#61; optional&#40;bool&#41;&#10;      values &#61; optional&#40;list&#40;string&#41;&#41;&#10;    &#125;&#41;&#41;&#10;    enforce &#61; optional&#40;bool&#41; &#35; for boolean policies only.&#10;    condition &#61; optional&#40;object&#40;&#123;&#10;      description &#61; optional&#40;string&#41;&#10;      expression  &#61; optional&#40;string&#41;&#10;      location    &#61; optional&#40;string&#41;&#10;      title       &#61; optional&#40;string&#41;&#10;    &#125;&#41;, &#123;&#125;&#41;&#10;  &#125;&#41;&#41;, &#91;&#93;&#41;&#10;&#125;&#41;&#41;">map&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [org_policies_data_path](variables.tf#L193) | Path containing org policies in YAML format. | <code>string</code> |  | <code>null</code> |
| [oslogin](variables.tf#L199) | Enable OS Login. | <code>bool</code> |  | <code>false</code> |
| [oslogin_admins](variables.tf#L205) | List of IAM-style identities that will be granted roles necessary for OS Login administrators. | <code>list&#40;string&#41;</code> |  | <code>&#91;&#93;</code> |
| [oslogin_users](variables.tf#L213) | List of IAM-style identities that will be granted roles necessary for OS Login users. | <code>list&#40;string&#41;</code> |  | <code>&#91;&#93;</code> |
| [parent](variables.tf#L220) | Parent folder or organization in 'folders/folder_id' or 'organizations/org_id' format. | <code>string</code> |  | <code>null</code> |
| [prefix](variables.tf#L230) | Optional prefix used to generate project id and name. | <code>string</code> |  | <code>null</code> |
| [project_create](variables.tf#L240) | Create project. When set to false, uses a data source to reference existing project. | <code>bool</code> |  | <code>true</code> |
| [service_config](variables.tf#L246) | Configure service API activation. | <code title="object&#40;&#123;&#10;  disable_on_destroy         &#61; bool&#10;  disable_dependent_services &#61; bool&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code title="&#123;&#10;  disable_on_destroy         &#61; false&#10;  disable_dependent_services &#61; false&#10;&#125;">&#123;&#8230;&#125;</code> |
| [service_encryption_key_ids](variables.tf#L258) | Cloud KMS encryption key in {SERVICE => [KEY_URL]} format. | <code>map&#40;list&#40;string&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [service_perimeter_bridges](variables.tf#L265) | Name of VPC-SC Bridge perimeters to add project into. See comment in the variables file for format. | <code>list&#40;string&#41;</code> |  | <code>null</code> |
| [service_perimeter_standard](variables.tf#L272) | Name of VPC-SC Standard perimeter to add project into. See comment in the variables file for format. | <code>string</code> |  | <code>null</code> |
| [services](variables.tf#L278) | Service APIs to enable. | <code>list&#40;string&#41;</code> |  | <code>&#91;&#93;</code> |
| [shared_vpc_host_config](variables.tf#L284) | Configures this project as a Shared VPC host project (mutually exclusive with shared_vpc_service_project). | <code title="object&#40;&#123;&#10;  enabled          &#61; bool&#10;  service_projects &#61; optional&#40;list&#40;string&#41;, &#91;&#93;&#41;&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code>null</code> |
| [shared_vpc_service_config](variables.tf#L293) | Configures this project as a Shared VPC service project (mutually exclusive with shared_vpc_host_config). | <code title="object&#40;&#123;&#10;  host_project         &#61; string&#10;  service_identity_iam &#61; optional&#40;map&#40;list&#40;string&#41;&#41;, &#123;&#125;&#41;&#10;  service_iam_grants   &#61; optional&#40;list&#40;string&#41;, &#91;&#93;&#41;&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code title="&#123;&#10;  host_project &#61; null&#10;&#125;">&#123;&#8230;&#125;</code> |
| [skip_delete](variables.tf#L315) | Allows the underlying resources to be destroyed without destroying the project itself. | <code>bool</code> |  | <code>false</code> |
| [tag_bindings](variables.tf#L321) | Tag bindings for this project, in key => tag value id format. | <code>map&#40;string&#41;</code> |  | <code>null</code> |

## Outputs

| name | description | sensitive |
|---|---|:---:|
| [custom_roles](outputs.tf#L17) | Ids of the created custom roles. |  |
| [id](outputs.tf#L25) | Project id. |  |
| [name](outputs.tf#L44) | Project name. |  |
| [number](outputs.tf#L56) | Project number. |  |
| [project_id](outputs.tf#L75) | Project id. |  |
| [service_accounts](outputs.tf#L94) | Product robot service accounts in project. |  |
| [services](outputs.tf#L110) | Service APIs to enabled in the project. |  |
| [sink_writer_identities](outputs.tf#L119) | Writer identities created for each sink. |  |
<!-- END TFDOC -->
