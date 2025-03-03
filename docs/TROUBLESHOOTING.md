# Troubleshooting

## Terminology

See [GLOSSARY.md](./GLOSSARY.md).

- - -

## Problems

- [Common issues](#common-issues)
- [Caller does not have permission in the Organization](#Caller-does-not-have-permission-in-the-organization)
- [Billing quota exceeded](#Billing-quota-exceeded)

- - -

## Common issues

- [Project quota exceeded](#project-quota-exceeded)
- [Default branch setting](#default-branch-setting)
- [Terraform State Snapshot lock](#terraform-state-snapshot-lock)
- [Application authenticated using end user credentials](#application-authenticated-using-end-user-credentials)
- [Cannot assign requested address error in Cloud Shell](#cannot-assign-requested-address-error-in-cloud-shell)
- [Error: Unsupported attribute](#error-unsupported-attribute)

- - -

### Project quota exceeded

**Error message:**

```
Error code 8, message: The project cannot be created because you have exceeded your allotted project quota
```

**Cause:**

This message means you have reached your [project creation quota](https://support.google.com/cloud/answer/6330231).

**Solution:**

In this case, you can use the [Request Project Quota Increase](https://support.google.com/code/contact/project_quota_increase)
form to request a quota increase.

In the support form,
for the field **Email addresses that will be used to create projects**,
use the email address of `terraform_service_account` that is created by the Terraform Example Foundation 0-bootstrap step.

**Notes:**

- If you see other quota errors, see the [Quota documentation](https://cloud.google.com/docs/quota).

### Default branch setting

**Error message:**

```
error: src refspec master does not match any
```
**Cause:**

This could be due to init.defaultBranch being set to something other than
`main`.

**Solution:**

1. Determine your default branch:
   ```
   git config init.defaultBranch
   ```
   Outputs `main` if you are in the main branch.
1. If your default branch is not set to `main`, set it:
   ```
   git config --global init.defaultBranch main
   ```

### Terraform State Snapshot lock

**Error message:**

When running the build for the branch `production` in step 3-networks in your **Foundation CI/CD Pipeline** the build fails with:

```
state snapshot was created by Terraform v1.x.x, which is newer than current v1.0.0; upgrade to Terraform v1.x.x or greater to work with this state
```

**Cause:**

The manual deploy step for the shared environment in [3-networks](../3-networks#deploying-with-cloud-build) was executed with a Terraform version newer than version v1.0.0 used in the **Foundation CI/CD Pipeline**.

**Solution:**

You have two options:

#### Downgrade your local Terraform version

You will need to re-run the deploy of the 3-networks shared environment with Terraform v1.0.0.

Steps:

- Go to folder `gcp-networks/envs/shared/`.
- Update `backend.tf` with your bucket name from the 0-bootstrap step.
- Run `terraform destroy` in the folder using the Terraform v1.x.x version.
- Delete the Terraform state file in `gs://YOUR-TF-STATE-BUCKET/terraform/networks/envs/shared/default.tfstate`. This bucket is in your **Seed Project**.
- Install Terraform v1.0.0.
- Re-run the manual deploy of 3-networks shared environment using Terraform v1.0.0.

#### Upgrade your 0-bootstrap runner image Terraform version

Replace `1.x.x` with the actual version of your local Terraform version in the following instructions:

- Go to folder `0-bootstrap`.
- Edit the local `terraform_version` in the Terraform [cb.tf](../0-bootstrap/cb.tf) file:
  - Upgrade loca `terraform_version` from `"1.0.0"` to `"1.x.x"`
- Run `terraform init`.
- Run `terraform plan` and review the output.
- Run `terraform apply`.

### Application authenticated using end user credentials

**Error message:**

When running `gcloud` commands in Cloud Shell like

```
gcloud scc notifications describe <scc_notification_name> --organization YOUR_ORGANIZATION_ID
```

or

```
gcloud access-context-manager policies list --organization YOUR_ORGANIZATION_ID --format="value(name)"
```

you receive the error:

```
Error 403: Your application has authenticated using end user credentials from the Google Cloud SDK or Google Cloud Shell which are not supported by the X.googleapis.com.
We recommend configuring the billing/quota_project setting in gcloud or using a service account through the auth/impersonate_service_account setting.
For more information about service accounts and how to use them in your application, see https://cloud.google.com/docs/authentication/.
```

**Cause:**

When using application default credentials in Cloud Shell a billing project is not available for APIs like `securitycenter.googleapis.com` or `accesscontextmanager.googleapis.com`.

**Solution:**

you can re-run the command using impersonation or providing a billing project:

- Impersonate the Terraform Service Account

```
--impersonate-service-account=terraform-org-sa@<SEED_PROJECT_ID>.iam.gserviceaccount.com
```

- Provide a billing project

```
--billing-project=<A-VALID-PROJECT-ID>
```

If you provide a billing project, you must have the `serviceusage.services.use` permission on the billing_project.

### Cannot assign requested address error in Cloud Shell

**Error message:**

When using [Google Cloud Shell](https://cloud.google.com/shell/docs) to deploy the code in ths repository, you may face an error like

```
dial tcp [2607:f8b0:400c:c15::5f]:443: connect: cannot assign requested address
```

when Terraform calls the Google APIs.

**Cause:**

This is a [known terraform issue](https://github.com/hashicorp/terraform-provider-google/issues/6782) regrading IPv6.

**Solution:**

At this time the alternatives are:

1. To use a [workaround](https://stackoverflow.com/a/62827358) to force Google API calls in Cloud Shell to use an IP from the `private.googleapis.com` range (199.36.153.8/30 ) or
1. To deploy the foundation code from a local machine that supports IPv6.

If you use the workaround, the API list should include the ones that are [allowed](../policy-library/policies/constraints/serviceusage_allow_basic_apis.yaml) in the terraform-example-foundation policy library.

### Error: Unsupported attribute

**Error message:**

```
Error: Unsupported attribute

  on main.tf line 22, in locals:
  22:   org_id               = data.terraform_remote_state.bootstrap.outputs.common_config.org_id
    ├────────────────
    │ data.terraform_remote_state.bootstrap.outputs is object with no attributes

This object does not have an attribute named "common_config".
```

**Cause:**

The stages after `0-bootstrap` use `terraform_remote_state` data source to read common configuration like the organization ID from the output of the `0-bootstrap` stage.
The error means that the Terraform state of the `0-bootstrap` stage was not copied to the Terraform state bucket created in stage `0-bootstrap`.

**Solution:**

Follow the instructions at the end of the [Deploying with Cloud Build](../0-bootstrap/README.md#deploying-with-cloud-build) section in the `0-bootstrap` README to copy the Terraform state to the Cloud Storage bucket created in stage `0-bootstrap` and retry planning/applying the stage you are deploying.

- - -

### Caller does not have permission in the Organization

**Error message:**

```
Error: Error when reading or editing Organization Not Found : <organization-id>: googleapi: Error 403: The caller does not have permission, forbidden
```

**Cause:**

User running Terraform is missing [Organization Administrator](https://cloud.google.com/iam/docs/understanding-roles#resource-manager-roles) predefined role at the Organization level.

**Solution:**

- If the user **does not have the role Organization Administrator** try the following:

You will need to request the roles to be granted to your user by your organization administration team.

- If the user **does have the role Organization Administrator** try the following:

```
gcloud auth application-default login
gcloud auth list # <- confirm that correct account has a star next to it
```

Re-run `terraform` after.

### Billing quota exceeded

**Error message:**

```
Error: Error setting billing account "XXXXXX-XXXXXX-XXXXXX" for project "projects/some-project": googleapi: Error 400: Precondition check failed., failedPrecondition
```

**Cause:**

Most likely this is related to a billing quota issue.

**Solution:**

try

```
gcloud alpha billing projects link projects/some-project --billing-account XXXXXX-XXXXXX-XXXXXX
```

If output states `Cloud billing quota exceeded`, you can use the [Request Billing Quota Increase](https://support.google.com/code/contact/billing_quota_increase) form to request a billing quota increase.
