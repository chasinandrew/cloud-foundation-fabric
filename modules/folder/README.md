# Google Cloud Folder Module

This module allows the creation and management of folders, including support for IAM bindings, organization policies, and hierarchical firewall rules.

<!-- BEGIN TOC -->
- [Basic example with IAM bindings](#basic-example-with-iam-bindings)
- [IAM](#iam)
- [Organization policies](#organization-policies)
  - [Organization Policy Factory](#organization-policy-factory)
- [Hierarchical Firewall Policy Attachments](#hierarchical-firewall-policy-attachments)
- [Log Sinks](#log-sinks)
- [Data Access Logs](#data-access-logs)
- [Tags](#tags)
- [Files](#files)
- [Variables](#variables)
- [Outputs](#outputs)
<!-- END TOC -->

## Basic example with IAM bindings

```hcl
module "folder" {
  source = "./fabric/modules/folder"
  parent = "organizations/1234567890"
  name   = "Folder name"
  group_iam = {
    "cloud-owners@example.org" = [
      "roles/owner",
      "roles/resourcemanager.folderAdmin",
      "roles/resourcemanager.projectCreator"
    ]
  }
  iam = {
    "roles/owner" = ["user:one@example.org"]
  }
  iam_additive = {
    "roles/compute.admin"  = ["user:a1@example.org", "user:a2@example.org"]
    "roles/compute.viewer" = ["user:a2@example.org"]
  }
  iam_additive_members = {
    "user:am1@example.org" = ["roles/storage.admin"]
    "user:am2@example.org" = ["roles/storage.objectViewer"]
  }
}
# tftest modules=1 resources=9 inventory=iam.yaml
```

## IAM

There are three mutually exclusive ways at the role level of managing IAM in this module

- non-authoritative via the `iam_additive` and `iam_additive_members` variables, where bindings created outside this module will coexist with those managed here
- authoritative via the `group_iam` and `iam` variables, where bindings created outside this module (eg in the console) will be removed at each `terraform apply` cycle if the same role is also managed here
- authoritative policy via the `iam_policy` variable, where any binding created outside this module (eg in the console) will be removed at each `terraform apply` cycle regardless of the role

The authoritative and additive approaches can be used together, provided different roles are managed by each. The IAM policy is incompatible with the other approaches, and must be used with extreme care.

Some care must be taken with the `groups_iam` variable (and in some situations with the additive variables) to ensure that variable keys are static values, so that Terraform is able to compute the dependency graph.

## Organization policies

To manage organization policies, the `orgpolicy.googleapis.com` service should be enabled in the quota project.

```hcl
module "folder" {
  source = "./fabric/modules/folder"
  parent = "organizations/1234567890"
  name   = "Folder name"
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

See the [organization policy factory in the project module](../project#organization-policy-factory).

## Hierarchical Firewall Policy Attachments

Hierarchical firewall policies can be managed via the [`net-firewall-policy`](../net-firewall-policy/) module, including support for factories. Once a policy is available, attaching it to the organization can be done either in the firewall policy module itself, or here:

```hcl
module "firewall-policy" {
  source    = "./fabric/modules/net-firewall-policy"
  name      = "test-1"
  parent_id = module.folder.id
  # attachment via the firewall policy module
  # attachments = {
  #   folder-1 = module.folder.id
  # }
}

module "folder" {
  source = "./fabric/modules/folder"
  parent = "organizations/1234567890"
  name   = "Folder name"
  # attachment via the organization module
  firewall_policy_associations = {
    test-1 = module.firewall-policy.id
  }
}
# tftest modules=2 resources=3
```

## Log Sinks

```hcl
module "gcs" {
  source        = "./fabric/modules/gcs"
  project_id    = "my-project"
  name          = "gcs_sink"
  force_destroy = true
}

module "dataset" {
  source     = "./fabric/modules/bigquery-dataset"
  project_id = "my-project"
  id         = "bq_sink"
}

module "pubsub" {
  source     = "./fabric/modules/pubsub"
  project_id = "my-project"
  name       = "pubsub_sink"
}

module "bucket" {
  source      = "./fabric/modules/logging-bucket"
  parent_type = "project"
  parent      = "my-project"
  id          = "bucket"
}

module "folder-sink" {
  source = "./fabric/modules/folder"
  parent = "folders/657104291943"
  name   = "my-folder"
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
module "folder" {
  source = "./fabric/modules/folder"
  parent = "folders/657104291943"
  name   = "my-folder"
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
module "folder" {
  source = "./fabric/modules/folder"
  parent = "folders/657104291943"
  name   = "my-folder"
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

module "folder" {
  source = "./fabric/modules/folder"
  name   = "Test"
  parent = module.org.organization_id
  tag_bindings = {
    env-prod = module.org.tag_values["environment/prod"].id
    foo      = "tagValues/12345678"
  }
}
# tftest modules=2 resources=6 inventory=tags.yaml
```

<!-- TFDOC OPTS files:1 -->
<!-- BEGIN TFDOC -->
## Files

| name | description | resources |
|---|---|---|
| [iam.tf](./iam.tf) | IAM bindings, roles and audit logging resources. | <code>google_folder_iam_binding</code> · <code>google_folder_iam_member</code> · <code>google_folder_iam_policy</code> |
| [logging.tf](./logging.tf) | Log sinks and supporting resources. | <code>google_bigquery_dataset_iam_member</code> · <code>google_folder_iam_audit_config</code> · <code>google_logging_folder_exclusion</code> · <code>google_logging_folder_sink</code> · <code>google_project_iam_member</code> · <code>google_pubsub_topic_iam_member</code> · <code>google_storage_bucket_iam_member</code> |
| [main.tf](./main.tf) | Module-level locals and resources. | <code>google_compute_firewall_policy_association</code> · <code>google_essential_contacts_contact</code> · <code>google_folder</code> |
| [organization-policies.tf](./organization-policies.tf) | Folder-level organization policies. | <code>google_org_policy_policy</code> |
| [outputs.tf](./outputs.tf) | Module outputs. |  |
| [tags.tf](./tags.tf) | None | <code>google_tags_tag_binding</code> |
| [variables.tf](./variables.tf) | Module variables. |  |
| [versions.tf](./versions.tf) | Version pins. |  |

## Variables

| name | description | type | required | default |
|---|---|:---:|:---:|:---:|
| [contacts](variables.tf#L17) | List of essential contacts for this resource. Must be in the form EMAIL -> [NOTIFICATION_TYPES]. Valid notification types are ALL, SUSPENSION, SECURITY, TECHNICAL, BILLING, LEGAL, PRODUCT_UPDATES. | <code>map&#40;list&#40;string&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [firewall_policy_associations](variables.tf#L24) | Hierarchical firewall policies to associate to this folder, in association name => policy id format. | <code>map&#40;string&#41;</code> |  | <code>&#123;&#125;</code> |
| [folder_create](variables.tf#L31) | Create folder. When set to false, uses id to reference an existing folder. | <code>bool</code> |  | <code>true</code> |
| [group_iam](variables.tf#L37) | Authoritative IAM binding for organization groups, in {GROUP_EMAIL => [ROLES]} format. Group emails need to be static. Can be used in combination with the `iam` variable. | <code>map&#40;list&#40;string&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [iam](variables.tf#L44) | IAM bindings in {ROLE => [MEMBERS]} format. | <code>map&#40;list&#40;string&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [iam_additive](variables.tf#L51) | Non authoritative IAM bindings, in {ROLE => [MEMBERS]} format. | <code>map&#40;list&#40;string&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [iam_additive_members](variables.tf#L58) | IAM additive bindings in {MEMBERS => [ROLE]} format. This might break if members are dynamic values. | <code>map&#40;list&#40;string&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [iam_policy](variables.tf#L65) | IAM authoritative policy in {ROLE => [MEMBERS]} format. Roles and members not explicitly listed will be cleared, use with extreme caution. | <code>map&#40;list&#40;string&#41;&#41;</code> |  | <code>null</code> |
| [id](variables.tf#L71) | Folder ID in case you use folder_create=false. | <code>string</code> |  | <code>null</code> |
| [logging_data_access](variables.tf#L77) | Control activation of data access logs. Format is service => { log type => [exempted members]}. The special 'allServices' key denotes configuration for all services. | <code>map&#40;map&#40;list&#40;string&#41;&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [logging_exclusions](variables.tf#L92) | Logging exclusions for this folder in the form {NAME -> FILTER}. | <code>map&#40;string&#41;</code> |  | <code>&#123;&#125;</code> |
| [logging_sinks](variables.tf#L99) | Logging sinks to create for the organization. | <code title="map&#40;object&#40;&#123;&#10;  bq_partitioned_table &#61; optional&#40;bool&#41;&#10;  description          &#61; optional&#40;string&#41;&#10;  destination          &#61; string&#10;  disabled             &#61; optional&#40;bool, false&#41;&#10;  exclusions           &#61; optional&#40;map&#40;string&#41;, &#123;&#125;&#41;&#10;  filter               &#61; string&#10;  include_children     &#61; optional&#40;bool, true&#41;&#10;  type                 &#61; string&#10;&#125;&#41;&#41;">map&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [name](variables.tf#L129) | Folder name. | <code>string</code> |  | <code>null</code> |
| [org_policies](variables.tf#L135) | Organization policies applied to this folder keyed by policy name. | <code title="map&#40;object&#40;&#123;&#10;  inherit_from_parent &#61; optional&#40;bool&#41; &#35; for list policies only.&#10;  reset               &#61; optional&#40;bool&#41;&#10;  rules &#61; optional&#40;list&#40;object&#40;&#123;&#10;    allow &#61; optional&#40;object&#40;&#123;&#10;      all    &#61; optional&#40;bool&#41;&#10;      values &#61; optional&#40;list&#40;string&#41;&#41;&#10;    &#125;&#41;&#41;&#10;    deny &#61; optional&#40;object&#40;&#123;&#10;      all    &#61; optional&#40;bool&#41;&#10;      values &#61; optional&#40;list&#40;string&#41;&#41;&#10;    &#125;&#41;&#41;&#10;    enforce &#61; optional&#40;bool&#41; &#35; for boolean policies only.&#10;    condition &#61; optional&#40;object&#40;&#123;&#10;      description &#61; optional&#40;string&#41;&#10;      expression  &#61; optional&#40;string&#41;&#10;      location    &#61; optional&#40;string&#41;&#10;      title       &#61; optional&#40;string&#41;&#10;    &#125;&#41;, &#123;&#125;&#41;&#10;  &#125;&#41;&#41;, &#91;&#93;&#41;&#10;&#125;&#41;&#41;">map&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> |  | <code>&#123;&#125;</code> |
| [org_policies_data_path](variables.tf#L162) | Path containing org policies in YAML format. | <code>string</code> |  | <code>null</code> |
| [parent](variables.tf#L168) | Parent in folders/folder_id or organizations/org_id format. | <code>string</code> |  | <code>null</code> |
| [tag_bindings](variables.tf#L178) | Tag bindings for this folder, in key => tag value id format. | <code>map&#40;string&#41;</code> |  | <code>null</code> |

## Outputs

| name | description | sensitive |
|---|---|:---:|
| [folder](outputs.tf#L17) | Folder resource. |  |
| [id](outputs.tf#L22) | Fully qualified folder id. |  |
| [name](outputs.tf#L32) | Folder name. |  |
| [sink_writer_identities](outputs.tf#L37) | Writer identities created for each sink. |  |
<!-- END TFDOC -->
