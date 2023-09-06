## 1. Introduction

### 1.1. How does GitLab SaaS integration with Conjur Cloud using JWT work?

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/3e22932a-9619-4f58-997b-76751e3feb52)

① Every GitLab CI/CD pipeline has a `CI_JOB_JWT_V2` JSON web token in the [predefined variables](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html)

- Example JWT

```json
{
  "namespace_id": "59407538",
  "namespace_path": "joetanx",
  "project_id": "46285066",
  "project_path": "joetanx/aws-cli-demo",
  "user_id": "12855715",
  "user_login": "joetanx",
  "user_email": "joe.tan@cyberark.com",
  "pipeline_id": "877636997",
  "pipeline_source": "push",
  "job_id": "4343330528",
  "ref": "main",
  "ref_type": "branch",
  "ref_path": "refs/heads/main",
  "ref_protected": "true",
  "jti": "f9fe5035-17f7-4259-b6f9-873fed0f57ea",
  "iss": "gitlab.com",
  "iat": 1684936926,
  "nbf": 1684936921,
  "exp": 1684940526,
  "sub": "job_4343330528"
}
```

> [!Note]
>
> The value of `CI_JOB_JWT_V2` is masked on GitLab jobs
> 
> To get the value:
> 
> - use `echo ${CI_JOB_JWT_V2} | base64`
> 
> - [decode](https://www.base64decode.org/) the base64 output
> 
> - then [decode](https://jwt.io/) the JWT

② The GitLab runner sends an authentication request to Conjur using REST API to the JWT authenticator URI (`https://<subdomain>.secretsmgr.cyberark.cloud/api/authn-jwt/<service-id>`)

- The URI for this demo is `https://apj-secrets.secretsmgr.cyberark.cloud/api/authn-jwt/jtan-gitlab`

③ Conjur fetches the public key from the GitLab JWKS URI

- The GibLab SaaS JWKS URI is at `https://gitlab.com/-/jwks/`
- This JWKS URI is set in the `jwks-uri` variable of the JWT authenticator in Conjur so that Conjur knows where to find the JWKS

④ Conjur verifies that the token is legit with the JWKS public key and authenticates application identity

- Conjur identifies the application identity via the `token-app-property` variable of the JWT authenticator
- The `token-app-property` variable is set in Conjur as the `project_path` claim in this demo
- Conjur further verifies the applications details as configured in the `annotations` listed in the `host` (application identity) declaration
- The annotations can be any claims on the JWT that do not change on a per-run basis, e.g. `namespace_id`, `namespace_path`, `project_path`

⑤ Conjur returns an access token to the GitLab runner if authentication is successful

⑥ The GitLab runner will then use the access token to retrieve the secrets using REST API to the secrets URI

### 1.2. GitLab free account and GitLab SaaS runners

GitLab free tier entitles [400 units of compute per month](https://about.gitlab.com/pricing/) to run [GitLab SaaS runners](https://docs.gitlab.com/ee/ci/runners/saas/linux_saas_runner.html) 

Credit card information is required to use GitLab SaaS runners

![image](https://github.com/joetanx/cjc-gitlab-terraform/assets/90442032/edee14c1-31e7-45e7-8ee6-35cd58f34ea3)

The `small` [machine type is the default](https://docs.gitlab.com/ee/ci/runners/saas/linux_saas_runner.html#machine-types-available-for-private-projects-x86-64) and [consumes compute units at 1 CI/CD minute cost factor rate](https://docs.gitlab.com/ee/ci/pipelines/cicd_minutes.html#cost-factor)

## 2. Preparation

### 2.1. AWS account

The jobs to be run on GitLab uses an AWS Secret Access Key

The IAM role will need to have the permissions for [S3 access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-iam-awsmanpol.html) and [access key rotation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html)

### 2.2. Onboard AWS account to Privilege Cloud

Ref: https://docs.cyberark.com/Product-Doc/OnlineHelp/PAS/Latest/en/Content/PASIMP/AmazonWebServicesAccessKeys.htm

![image](https://github.com/joetanx/cjc-gitlab-terraform/assets/90442032/30ca1a08-c8ab-4be1-a7e9-58ed3d3098b0)

### 2.3. Configure account sync from Privilege Cloud to Conjur Cloud

Ref: https://docs.cyberark.com/Product-Doc/OnlineHelp/ConjurCloud/Latest/en/Content/HomeTilesLPs/LP-Tile4.htm

![image](https://github.com/joetanx/cjc-gitlab-terraform/assets/90442032/32ccdc75-0b21-4497-9910-37c66a58b935)

![image](https://github.com/joetanx/cjc-gitlab-terraform/assets/90442032/e7243f2f-f8db-4c7b-98b2-97d31becf11b)

![image](https://github.com/joetanx/cjc-gitlab-terraform/assets/90442032/b38997b4-a1d1-41d2-a0cf-734de533828f)

![image](https://github.com/joetanx/cjc-gitlab-terraform/assets/90442032/de72e937-a15e-462d-846d-4ce88cdbff17)

## 3. Conjur policies for GitLab JWT

## 3.1. Details of Conjur policies used in this demo

#### 3.1.1. gitlab-hosts.yaml

- `jtan/gitlab` - policy name, this forms the `identity-path` of the app IDs
- applications `joetanx/aws-cli-demo`, `joetanx/terraform-aws-s3-demo` and `joetanx/terraform-aws-s3-cleanup` are configured
  - the `id` of the `host` corresponds to the `token-app-property`
  - annotations of the `host` are optional and corresponds to claims in the JWT token claims - the more specific the annotations/claims configured, the more precise and secure the application authentication
- the host layer is granted as a member of the `vault/jtan/delegation/consumers` group to authorize access to the AWS secret access key synchronized from Privilege Cloud

#### 3.1.2. authn-jwt-gitlab.yaml

- Configures the JWT authenticator (https://docs.cyberark.com/Product-Doc/OnlineHelp/ConjurCloud/Latest/en/Content/Operations/Services/cjr-authn-jwt-uc.htm)
- Defines the authenticator webservice at `authn-jwt/jtan-gitlab`
  - The format of the authenticator webservice is `authn-jwt/<service-id>`, the `<service-id>` used in this demo is `jtan-gitlab`, this is the URI where the GitLab pipeline will authenticate to.

- Defines the authentication variables: how the JWT Authenticator gets the signing keys

| Variables | Description |
|---|---|
| `jwks-uri` | JSON Web Key Set (JWKS) URI. For GitLab this is `https://gitlab.com/-/jwks/`. |
| `token-app-property` | The JWT claim to be used to identify the application. This demo uses the `project_path` claim from GitLab.  |
| `identity-path` | The Conjur policy path where the `host`s are defined in Conjur policy. The `host`s in `gitlab-hosts.yaml` are created under `jtan/gitlab`, so the `identity-path` is `data/jtan/gitlab`. |
| `issuer` | URI of the JWT issuer. This is the GitLab URL. This is included in `iss` claim in the JWT token claims. |

- Defines `consumers` group - applications that are authorized to authenticate using this JWT authenticator are added to this group
- Defines `operators` group - users who are authorized to check the status of this JWT authenticator are added to this group

## 3.2. Load the Conjur policies and prepare Conjur for GitLab JWT

Login to Conjur:

```console
conjur init -u https://<subdomain>.secretsmgr.cyberark.cloud/api
conjur login -i <username> -p <password>
```

Download and load the Conjur policies:

```console
curl -sLO https://github.com/joetanx/cjc-gitlab/raw/main/authn-jwt-gitlab.yaml
curl -sLO https://github.com/joetanx/cjc-gitlab/raw/main/gitlab-hosts.yaml
conjur policy load -b conjur/authn-jwt -f authn-jwt-gitlab.yaml
conjur policy load -b data -f gitlab-hosts.yaml
```

Enable the JWT Authenticator:

```console
conjur authenticator enable --id authn-jwt/jtan-gitlab
```

Populate the variables:

```console
conjur variable set -i conjur/authn-jwt/jtan-gitlab/jwks-uri -v https://gitlab.com/-/jwks/
conjur variable set -i conjur/authn-jwt/jtan-gitlab/token-app-property -v project_path
conjur variable set -i conjur/authn-jwt/jtan-gitlab/identity-path -v data/jtan/gitlab
conjur variable set -i conjur/authn-jwt/jtan-gitlab/issuer -v https://gitlab.com
```

## 4. GitLab Projects

### 4.1. AWS CLI Demo

This project tests retrieval of the AWS secret access key from Conjur

#### 4.1.1. Create a new project

GitLab project name: `AWS CLI Demo`

☝️ Project name is important! The `project path` must match the `host` identity configured in the Conjur policy

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/d1bb1f4c-fdab-4d91-bbda-be14b9fdd941)

#### 4.1.2. Create the [main.tf](/aws-cli-demo/main.tf) file

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/49a989a5-4d58-41cb-a018-30cc73546759)

#### 4.1.3. Edit the GitLab CI/CD file

There are 2 stages in the pipeline code below:
1. Fetch variables from Conjur (using CyberArk GitLab runner image)
  - Authenticate to Conjur `authn-jwt/jtan-gitlab` using `CI_JOB_JWT_V2`
  - Retrive AWS credentials
  - Pass the credentials to the next stage using `artifacts:`, `reports:`, `dotenv:`
2. Test the AWS credentials
  - Run Terraform using `docker.io/hashicorp/terraform:latest` image
  - Run AWS CLI using `docker.io/amazon/aws-cli:latest` image

https://github.com/joetanx/cjc-gitlab/blob/9de34850e90a07ff595f03dd9906175b114a77fa/aws-cli-demo/.gitlab-ci.yml#L1-L32

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/c76dfa39-59a9-4398-81d2-6558daaa97fd)

#### 4.1.4. Pipeline run results

All jobs passed in the pipeline:

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/b7c7ff9f-1851-4f71-b6ba-48b4a9258ad0)

Output for fetch variables job:

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/7aea7310-56d4-4623-a5f2-58ca0ad6f761)

Output for Terraform job:

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/af489be6-3422-4000-8b4e-8ae6eb4651c2)

Output for AWS CLI job:

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/34bf619a-855b-4297-8736-3f951d085ecf)

### 4.2. Terraform AWS S3 Demo

This project demostrates the use of AWS secret access key retrieved from Conjur to create a S3 bucket

#### 4.2.1. Create a new project

GitLab project name: `Terraform AWS S3 Demo`

☝️ Project name is important! The `project path` must match the `host` identity configured in the Conjur policy

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/68601f6e-484c-4bd2-b0b1-bcdfbf968938)

#### 4.2.2. Create the [demo.txt](/terraform-aws-s3-demo/demo.txt), [main.tf](/terraform-aws-s3-demo/main.tf) and [provider.tf](/terraform-aws-s3-demo/provider.tf) files

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/ef002a6d-29b0-4b72-a6ed-8b10cf723bf9)

#### 4.2.3. Edit the GitLab CI/CD file

There are 2 stages in the pipeline code below:
1. Fetch variables from Conjur (using CyberArk GitLab runner image)
  - Authenticate to Conjur `authn-jwt/jtan-gitlab` using `CI_JOB_JWT_V2`
  - Retrive AWS credentials
  - Pass the credentials to the next stage using `artifacts:`, `reports:`, `dotenv:`
2. Run Terraform to create S3 bucket according to `main.tf` using credentials from Conjur

https://github.com/joetanx/cjc-gitlab/blob/49bdde52830178890f7e0f3dfff91f2fce6d0989/terraform-aws-s3-demo/.gitlab-ci.yml#L1-L25

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/c5b69506-c149-4a92-a532-f4cf34ef7138)

#### 4.2.4. Pipeline run results

Both jobs passed in the pipeline:

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/e653339b-d8eb-467a-94f3-fd264cac3bb3)

Output for fetch variables job:

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/b9804cdf-9ca3-4045-83a6-c9a76a133ab2)

Output for Terraform job:

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/fda5bfd7-6af2-4f7d-9e39-8ff0349b10fe)

### 4.3. Terraform AWS S3 Cleanup

This project is used to verify the bucket created from the demo job above

#### 4.3.1. Create a new project

GitLab project name: `Terraform AWS S3 Cleanup`

☝️ Project name is important! The `project path` must match the `host` identity configured in the Conjur policy

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/2e811dea-2cce-4ff4-afe9-e073c1943d5f)

#### 4.3.2. Edit the GitLab CI/CD file

There are 3 stages in the pipeline code below:
1. Fetch variables from Conjur (using CyberArk GitLab runner image)
  - Authenticate to Conjur `authn-jwt/jtan-gitlab` using `CI_JOB_JWT_V2`
  - Retrive AWS credentials
  - Pass the credentials to the next stage using `artifacts:`, `reports:`, `dotenv:`
2. Run AWS CLI to get the demo file from the S3 bucket created above using credentials from Conjur
3. Manual job to delete the bucket

https://github.com/joetanx/cjc-gitlab/blob/a883932da82feb94b8cdd2e65f2dd572add83500/terraform-aws-s3-cleanup/.gitlab-ci.yml#L1-L34

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/8df6cdf4-4732-4112-8f75-b0ce0de9d358)

#### 4.2.3. Pipeline run results

Both jobs passed in the pipeline (manual job is pending manual activation):

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/d7cb5e6e-88de-425e-b37d-1bc9cb111390)

Output for fetch variables job:

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/7e01db80-e84a-4ac4-a77c-9faaab737ea5)

Output for AWS CLI job:

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/2fc2cafc-ff89-4e54-8ad7-978db2c4f8cc)

Proceed to run the last job to delete bucket:

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/f4c9eecd-1066-4488-9e68-1256861fe6a8)

Output for delete bucket job:

![image](https://github.com/joetanx/cjc-gitlab/assets/90442032/cd42e890-81d8-4653-b1df-27c095bed536)

## 5. Audit Events

[Activities](https://docs.cyberark.com/Product-Doc/OnlineHelp/ConjurCloud/Latest/en/Content/Audit/isp_system-activities.htm) in Conjur Cloud can be viewed on CyberArk Audit where details of the action (e.g. authenicate, fetch) and the host identities are recorded

![image](https://github.com/joetanx/cjc-gitlab-terraform/assets/90442032/837f69f0-077d-42e1-9f31-598b0ac3f447)
