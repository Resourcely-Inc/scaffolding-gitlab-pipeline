# Scaffolding GitLab CI/CD Pipeline

This repository demonstrates how to integrate the [Resourcely guardrail validation](https://github.com/Resourcely-Inc/resourcely-gitLab-template) job into a repository using GitLab CI/CD. It runs Terraform using the official [Hashicorp Terraform Docker Images](https://github.com/hashicorp/terraform).

## Assumption

This repository uses GitLab CI/CD to run terraform plan. Once a plan is downloaded to a designated directory, the Resourcely guardrail validation job runs on the configured path. If you use a different runner, see the scaffolding repository for that runner:

- [GitHub Actions](https://github.com/Resourcely-Inc/scaffolding-github-actions)
- [GitHub Actions + Terraform Cloud](https://github.com/Resourcely-Inc/scaffolding-github-terraform-cloud)
- [GitLab + Terraform Cloud](https://github.com/Resourcely-Inc/scaffolding-gitlab-pipeline-terraform-cloud)

## Prerequisites

1. [A Resourcely Account](https://docs.resourcely.io/resourcely-terms/user-management/resourcely-account)
2. [Resourcely GitLab SCM Configured](https://docs.resourcely.io/integrations/source-code-management/gitlab)
3. [GitLab Premium or Ultimate subscription](https://about.gitlab.com/pricing/)
4. [Maintainer Role or Higher](https://docs.gitlab.com/ee/user/permissions.html#roles) in the GitLab project
5. [AWS Provider Credentials](https://registry.terraform.io/providers/hashicorp/aws/latest/docs#authentication-and-configuration)

## Usage

1. [Import this project to your GitLab group by URL](https://docs.gitlab.com/ee/user/project/import/repo_by_url.html)  
    a. On the left sidebar, at the top, select **Create new (+)** and **New project/repository**  
    b. Select **Import project**  
    c. Select **Repository by URL**  
    d. Enter the **Git repository URL**: https://github.com/Resourcely-Inc/scaffolding-gitlab-pipeline.git  
    e. Complete the remaining fields  
    f. Select **Create project**
2. [Allow Resourcely to monitor the newly created project](https://docs.resourcely.io/onboarding/source-code-management-integration/gitlab#choosing-a-repository-for-resourcely-to-monitor)
3. [Generate a Resourcely API Token](https://docs.resourcely.io/onboarding/api-access-token-generation) and save it in a safe place
4. Add your Resourcely API Token to your [GitLab project CI/CD variables](https://docs.gitlab.com/ee/ci/variables/)  
    a. Go to the GitLab project that Resourcely will validate  
    b. In the side tab, navigate to **Settings > CI/CD**  
    c. Expand the **Variables** tab  
    d. Click the **Add variable** button  
    e. Add the `RESOURCELY_API_TOKEN` as the key and the token as the value
    f. Evaluate whether to unselect **Protect variable**, depending on the need to use the token in un-protected branches, while considering security implications  
    g. Select the **Mask variable** to protect sensitive data from being seen in job logs  
    h. Unselect **Expand variable reference**  
    i. Press the **Add variable** button  
5. Add your AWS credentials `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` to GitLab following the same process in step 3
6. [Provision Infrastructure using Resourcely](https://docs.resourcely.io/using-resourcely)

Once a new Resource has been created via Merge-Request, the Resourcely job will automatically kick-off. It runs in the **test** stage by default.

## How it works

When a merge-request is created using Resourcely:

1. GitLab CI kicks off the `validate` stage  
    a. The `hashicorp/terraform:light` container image is loaded  
    b. `terraform init` is run to initialize a working directory containing Terraform configuration files  
    c. `terraform validate` is run to validate the Terraform configuration files  
2. After `validate` stage completes, GitLab CI kicks off the `plan` stage  
    a. The `hashicorp/terraform:light` container image is loaded  
    b. `terraform init` is run to initialize a working directory containing Terraform configuration files  
    c. `terraform plan` is run to create an execution plan  
    d. `terraform show` is run to download the plan as a json  
    e. The plan json is stored as a GitLab artifact in the `$TF_PLAN_DIRECTORY`  
3. After the `plan` stage completes, GitLab CI kicks off the `test` stage  
    a. The `test` stage is loaded by the [Resourcely template](https://gitlab.com/fern-inc/resourcely/resourcely-gitlab-guardrails) that is included in this project's `.gitlab-ci.yml` 
    b. The `ghcr.io/resourcely-inc/resourcely-cli:$RESOURCELY_IMAGE` container image is loaded  
    c. The `resourcely_guardrails` job runs `resourcely-cli evaluate` scanning the Terraform plan json(s)  
    d. The resources generated with Resourcely within the merge-request are validated against your Resourcely guardrails  
4. The `test` stage completes  
    a. If guardrail violations are found, Resourcely will assign a reviewer to the merge-request and require approval before it can be merged
