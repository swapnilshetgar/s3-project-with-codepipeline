# AWS Static Website CI/CD: Step-by-Step Guide

This guide turns the supplied architecture into a working static website delivery path. It assumes a GitHub repository containing a website (the included `index.html` is a starter), an AWS account, and a deployment region. Replace every `<PLACEHOLDER>` before running commands.

## Correct architecture

```text
Developer -> GitHub -> CodePipeline (Source -> Build -> Deploy) -> CodeBuild -> S3
                                                                  CloudFront -> WAF -> visitors
ACM certificate (us-east-1 for CloudFront) ----------------------> CloudFront
Route 53 alias (optional) --------------------------------------> CloudFront
```

CodePipeline orchestrates the stages. CodeBuild installs dependencies if needed and produces the deployable website artifact. The pipeline's S3 deploy action copies that artifact directly into the website bucket. CodeDeploy is for deploying application revisions to supported compute targets; it is not required for this S3 static-site flow. CloudFront should use the S3 REST origin with Origin Access Control (OAC), so the bucket stays private. WAF attaches to CloudFront and filters viewer requests.

## 1. Prepare the repository

1. Create a GitHub repository and add the site files at its root, including `index.html` and `buildspec.yml` from this package.
2. For a plain HTML/CSS/JS site, no compilation is required. For a framework site, make `buildspec.yml` run its build command and set `base-directory` to its output folder (for example `dist`).
3. Commit and push to the branch that should trigger deployments (for example `main`).

## 2. Choose names and region

Pick a unique S3 bucket name such as `example.com-prod-site-<account-id>`, a deployment region, and (optionally) a domain such as `www.example.com`. Keep the bucket, CodeBuild project, and pipeline in the same region. CloudFront is global; its ACM certificate must be in **us-east-1**. If using WAF with CloudFront, create the Web ACL in the CloudFront/global scope (the console may show US East (N. Virginia)).

## 3. Create the private S3 origin bucket

In **S3 → Buckets → Create bucket**:

1. Select the pipeline region and a globally unique bucket name.
2. Keep **Block all public access** enabled and enable bucket versioning (recommended).
3. Create the bucket. Do not enable S3 static website hosting for this OAC design; CloudFront serves the private REST endpoint and its default root object.
4. Leave Object Ownership as **Bucket owner enforced** (ACLs disabled).

CLI equivalent (run in a shell with AWS CLI credentials for the intended account):

```bash
aws s3api create-bucket --bucket <BUCKET> --region <REGION> --create-bucket-configuration LocationConstraint=<REGION>
aws s3api put-public-access-block --bucket <BUCKET> --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws s3api put-bucket-versioning --bucket <BUCKET> --versioning-configuration Status=Enabled
```

For `us-east-1`, omit `--create-bucket-configuration LocationConstraint=...` from `create-bucket`.

## 4. Create the CloudFront distribution and lock the origin

Create CloudFront before finalizing the bucket policy, so you can use its distribution ID:

1. In **CloudFront → Distributions → Create distribution**, choose the S3 bucket's **regular bucket origin**, not its website endpoint.
2. Choose **Origin access → Origin access control settings → Create new OAC**; signing behavior should be **Sign requests (recommended)**.
3. Set **Viewer protocol policy** to **Redirect HTTP to HTTPS**, allowed methods to **GET, HEAD** (add OPTIONS only if needed), and **Default root object** to `index.html`.
4. Leave custom domain and certificate empty for now if you have not requested the certificate. Create the distribution.
5. In the distribution's origin settings, confirm the OAC is attached. Use **Copy policy** / **Copy policy statement** and apply the generated statement in **S3 → bucket → Permissions → Bucket policy**. This grants `cloudfront.amazonaws.com` `s3:GetObject` only for that distribution. Do not grant public access.

CLI policy template (substitute account, bucket and distribution ID; the console-generated policy is preferable):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontOACRead",
    "Effect": "Allow",
    "Principal": {"Service": "cloudfront.amazonaws.com"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::<BUCKET>/*",
    "Condition": {"StringEquals": {"AWS:SourceArn": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>"}}
  }]
}
```

Save as `bucket-policy.json`, then run `aws s3api put-bucket-policy --bucket <BUCKET> --policy file://bucket-policy.json`.

## 5. Request and attach the ACM certificate (custom domain only)

1. Switch the AWS console region to **US East (N. Virginia) / us-east-1**.
2. Open **ACM → Request certificate → Request a public certificate**. Add every hostname you plan to use (for example `www.example.com`; add `example.com` separately if both are needed).
3. Choose DNS validation. Create the suggested CNAME records in Route 53, or at your DNS provider, and wait for **Issued**.
4. Edit the CloudFront distribution's **Alternate domain names (CNAMEs)** and select this certificate. Select TLS policy `TLSv1.2_2021` or a newer supported policy, then save and wait for deployment.

Without a custom domain, use the CloudFront `d....cloudfront.net` address and skip ACM/Route 53.

## 6. Create the WAF web ACL (recommended)

1. Open **AWS WAF & Shield → Web ACLs → Create web ACL**. Set resource type to **Amazon CloudFront distributions** (global scope), name it, and choose the appropriate metrics/logging options.
2. Add AWS Managed Rules. A sensible starting point is **Core rule set** and **Known bad inputs**. Consider the **Amazon IP reputation list**. Managed rules can block legitimate requests; begin in **Count** mode where possible, review sampled requests/metrics, then switch to Block after tuning exclusions.
3. Set the default action to **Allow**, review rule order and capacity, and create the ACL.
4. Associate the web ACL with the CloudFront distribution (or select it in distribution security settings). Wait for distribution deployment.

CLI outline: create the ACL using `aws wafv2 create-web-acl --scope CLOUDFRONT --region us-east-1 ...`; then associate with `aws wafv2 associate-web-acl --web-acl-arn <ACL_ARN> --resource-arn <DISTRIBUTION_ARN> --region us-east-1`. The create command requires a complete rules JSON and a caller identity with WAF permissions; use the console for a first setup.

## 7. IAM roles and least privilege

Use separate service roles; do not use personal access keys in build settings.

**CodeBuild service role** (trusted service: `codebuild.amazonaws.com`):

- CloudWatch Logs: `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents` for the project's log group/streams.
- Artifact bucket: required S3 read/write access to the CodePipeline artifact bucket/prefix used by the project. The console-created role/policy is a reliable baseline; scope S3 resources to that bucket and its objects.
- KMS decrypt/encrypt only if the artifact bucket uses a customer-managed KMS key.
- GitHub source credentials should be supplied by CodePipeline's connection, not embedded in the build role.

**CodePipeline service role** (trusted service: `codepipeline.amazonaws.com`):

- Read/write artifact objects in the pipeline artifact bucket (including multipart upload actions as required by S3).
- CodeBuild: `codebuild:StartBuild`, `codebuild:BatchGetBuilds` on the specific project.
- Deploy bucket: `s3:PutObject`, `s3:PutObjectAcl` only if ACLs are enabled (not needed here), and `s3:ListBucket` on the destination bucket. With Bucket owner enforced, do not set an ACL.
- KMS permissions on the artifact key if applicable.
- GitHub connection: `codeconnections:UseConnection` on the specific connection ARN (older accounts may display `codestar-connections:UseConnection`).

**CloudFront OAC** uses the bucket policy in step 4; it does not need an IAM user. **Human setup/deployment administrator** needs permission to create/configure the listed resources and pass the CodeBuild/CodePipeline service roles (`iam:PassRole`). Avoid broad `*` policies in a production account. In the console, let CodePipeline create its role and artifact bucket, then narrow policies after a successful first run if your organization requires stricter controls.

## 8. Configure CodeBuild

1. In **CodeBuild → Build projects → Create build project**, choose a project name and **Managed image**, Linux, Standard runtime, and the current supported image.
2. Choose **GitHub** as source only if building independently; for this pipeline, the project is started by CodePipeline, so select **CodePipeline** as the source provider when available.
3. Select the CodeBuild service role. Set buildspec to **Use a buildspec file** at repository root.
4. Set artifacts to **CodePipeline**. Do not configure a separate S3 artifact destination here; CodePipeline receives the output and its S3 deploy action publishes it.
5. Add environment variables only for non-secret configuration. Store secrets in Secrets Manager/Parameter Store if needed.

Use the following root `buildspec.yml` for plain static files:

```yaml
version: 0.2
phases:
  build:
    commands:
      - echo "Preparing static website artifact"
      - test -f index.html
artifacts:
  files:
    - '**/*'
  discard-paths: no
```

For a framework, replace the build commands with its install/build commands and set `artifacts: base-directory: dist` (or the actual output directory) and `files: - '**/*'`. Pin dependencies with a lockfile.

## 9. Configure CodePipeline

1. In **CodePipeline → Create pipeline**, choose a pipeline name and **New service role**. Let the wizard create an artifact bucket in the same region.
2. **Source stage:** choose **GitHub (via GitHub App / CodeConnections)**, connect/authorize the GitHub App, select repository and branch, and enable change detection. The first connection may need to be completed from the AWS Developer Tools connection page.
3. **Build stage:** choose **AWS CodeBuild**, select the project from step 8, and add an output artifact name such as `BuildOutput`. Ensure CodeBuild's artifact type is CodePipeline.
4. **Deploy stage:** choose **Amazon S3**, set the private site bucket, select `BuildOutput`, and enable **Extract file before deploy**. Do not set a canned ACL.
5. Create the pipeline and wait for the first run. Check that the source, build, and deploy actions succeed.

Equivalent declarative stage shape (connection ARN/region, project, bucket and role ARNs must be supplied; create in the console if this is your first pipeline):

```json
{
  "pipeline": {
    "name": "static-site-pipeline",
    "roleArn": "arn:aws:iam::<ACCOUNT_ID>:role/<PIPELINE_ROLE>",
    "artifactStore": {"type": "S3", "location": "<ARTIFACT_BUCKET>"},
    "stages": [
      {"name":"Source","actions":[{"name":"GitHub","actionTypeId":{"category":"Source","owner":"AWS","provider":"CodeStarSourceConnection","version":"1"},"configuration":{"ConnectionArn":"<CONNECTION_ARN>","FullRepositoryId":"<OWNER>/<REPO>","BranchName":"main","DetectChanges":"true"},"outputArtifacts":[{"name":"SourceOutput"}],"runOrder":1}]},
      {"name":"Build","actions":[{"name":"Build","actionTypeId":{"category":"Build","owner":"AWS","provider":"CodeBuild","version":"1"},"configuration":{"ProjectName":"<CODEBUILD_PROJECT>"},"inputArtifacts":[{"name":"SourceOutput"}],"outputArtifacts":[{"name":"BuildOutput"}],"runOrder":1}]},
      {"name":"Deploy","actions":[{"name":"DeployToS3","actionTypeId":{"category":"Deploy","owner":"AWS","provider":"S3","version":"1"},"configuration":{"BucketName":"<BUCKET>","Extract":"true"},"inputArtifacts":[{"name":"BuildOutput"}],"runOrder":1}]}
    ]
  }
}
```

If using CLI, save a complete pipeline document as JSON and use `aws codepipeline create-pipeline --cli-input-json file://pipeline.json`; update with `update-pipeline`. The pipeline artifact bucket and KMS configuration must match the region and role permissions.

## 10. Optional Route 53 DNS

After ACM shows **Issued** and CloudFront is deployed:

1. In **Route 53 → Hosted zones**, open the public hosted zone for your domain.
2. Create an **A** record for the hostname (and **AAAA** if you want IPv6), choose **Alias**, and select the CloudFront distribution as the alias target. For a subdomain, use the subdomain as the record name. For the zone apex, Route 53 supports an alias; many other DNS providers do not.
3. If DNS is hosted elsewhere, create a CNAME for a subdomain pointing to the CloudFront hostname. A zone apex usually requires ALIAS/ANAME support.
4. Wait for DNS propagation and verify the hostname resolves to the distribution.

## 11. Deploy, invalidate, and verify

1. Push a small change to the configured GitHub branch. Watch **CodePipeline → execution history** for Source → Build → Deploy success.
2. Open the CloudFront domain. Confirm the page loads over HTTPS, `index.html` is the root page, and static assets load.
3. After a deployment, CloudFront can keep cached copies until TTL expiry. Prefer versioned filenames for assets. For immediate HTML refresh, create an invalidation for `/` and `/index.html` in **CloudFront → Invalidations**. CLI: `aws cloudfront create-invalidation --distribution-id <DISTRIBUTION_ID> --paths / /index.html`.
4. Verify the S3 bucket is still private, CloudFront has the correct OAC, and WAF metrics are visible. Test the custom domain if configured.

## 12. Troubleshooting

| Symptom | Check |
|---|---|
| CodePipeline cannot connect to GitHub | Finish the CodeConnections installation/authorization; confirm the connection status is **Available** and pipeline role has `UseConnection`. |
| CodeBuild says `index.html` missing | Confirm repository root and buildspec path; for framework output, configure the correct `base-directory`. |
| Build succeeds but deploy fails with AccessDenied | Check pipeline role S3 permissions on bucket and objects, correct bucket region, artifact permissions, and KMS access if encrypted. |
| CloudFront returns 403 | Check OAC is attached, bucket policy SourceArn distribution ID is exact, object key exists, bucket is private, and default root object is `index.html`. |
| CloudFront returns 404 for `/` | Set Default root object to `index.html`; verify deployment includes it at bucket root. |
| Website updates are not visible | Wait for pipeline/distribution propagation; invalidate `/index.html` and `/` or reduce HTML cache TTL. |
| HTTPS certificate unavailable in CloudFront | Certificate must be **Issued** in ACM **us-east-1**, and all alternate names must match certificate names. |
| Custom domain does not resolve | Confirm alias/CNAME target, distribution alternate name, certificate, public hosted zone/delegation, and DNS propagation. |
| WAF blocks valid visitors | Review sampled requests and CloudWatch metrics; change the specific managed rule to Count or add a narrow scope-down/exclusion, then retest. |

## Resource cleanup

To avoid ongoing charges, first disable/delete the pipeline, disassociate and delete the WAF ACL, then disable and delete the CloudFront distribution (wait for deployment). Empty and delete both the site bucket and artifact bucket, then remove CodeBuild, roles/connections if unused, and the ACM certificate after it is detached. Retain logs or versions if needed before deleting them.
