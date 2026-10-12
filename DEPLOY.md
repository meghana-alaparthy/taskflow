# Deploying Taskflow to AWS from GitHub ($0 plan)

Flow: `git push` to `main` → GitHub Actions builds + tests the Java app → uploads the jar to S3 → Elastic Beanstalk runs it.

Run locally: `./mvnw spring-boot:run` → http://localhost:5000

## Part 1 – AWS account (you do this; it needs a card)
1. Sign up at https://aws.amazon.com → "Create an AWS account".
2. **Choose the "Free plan"** when asked (not "Paid plan"). It uses promotional credits and can't bill you; if credits run out or the period (about 6 months) ends, the account is restricted until you upgrade. Check the terms shown at sign-up.
3. Sign in as root, turn on **MFA** (Account menu → Security credentials).
4. Billing → Budgets → Create budget → "Zero spend" template.
5. Pick one region (e.g. `us-east-1`) and use it everywhere.

## Part 2 – Create the Elastic Beanstalk app (console)
1. Search "Elastic Beanstalk" → **Create application**.
2. Name: `taskflow`. Tier: **Web server**. Platform: **Java** (Corretto 21, "Java SE"). Code: **Sample application**.
3. Preset: **Single instance** (NOT "High availability" – its load balancer costs money). Instance type **t3.micro** (or t2.micro if offered). No database, no custom domain.
4. Service role / EC2 instance profile: let the wizard create them, or create `aws-elasticbeanstalk-service-role` and `aws-elasticbeanstalk-ec2-role` (attach `AWSElasticBeanstalkWebTier`).
5. Wait ~5 min until health is green. Note the environment name (e.g. `taskflow-env`).

## Part 3 – S3 bucket for deploy bundles
S3 → Create bucket, e.g. `taskflow-deploys-<yourname>`; keep "Block all public access" ON.

## Part 4 – Let GitHub log in to AWS (OIDC, no secret keys)
1. IAM → Identity providers → Add provider → OpenID Connect: URL `https://token.actions.githubusercontent.com`, audience `sts.amazonaws.com`.
2. IAM → Roles → Create role → Web identity → that provider → GitHub user `meghana-alaparthy`, repo `taskflow`, branch `main`. Name it `github-actions-deploy`.
3. Attach this policy (replace BUCKET):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect":"Allow","Action":["s3:PutObject","s3:GetObject","s3:GetBucketLocation","s3:ListBucket"],"Resource":["arn:aws:s3:::BUCKET","arn:aws:s3:::BUCKET/*"]},
    {"Effect":"Allow","Action":["elasticbeanstalk:*","autoscaling:*","ec2:Describe*","cloudformation:*","elasticloadbalancing:Describe*"],"Resource":"*"}
  ]
}
```
   (Broad on purpose for a first run; tighten later.) Copy the role **ARN**.

## Part 5 – Tell GitHub (repo → Settings → Secrets and variables → Actions)
- Secret `AWS_ROLE_ARN` = role ARN
- Variables `AWS_REGION`, `EB_APP` (`taskflow`), `EB_ENV`, `EB_BUCKET`

## Part 6 – Deploy
Push to `main` (or Actions tab → Run workflow). Open the environment URL when it finishes.

Note: data lives in an in-memory H2 database, so tasks reset on every restart/deploy. That keeps it $0; move to RDS later if you need persistence.

## Cleanup (avoid charges)
Elastic Beanstalk → Environments → Actions → **Terminate environment**; empty and delete the S3 bucket.
