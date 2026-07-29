# Lab 2: Infrastructure as Code with AWS CDK

Check box if done: [ ]

## Overview
This repo already covers Terraform in depth ([SAA-C03 Lab 8](../SAA-C03/aws-lab-8-terraform-aws.md)) — provider config, state, modules. But "Terraform-only" is a gap on a resume for teams that standardize on AWS-native tooling: plenty of AWS-heavy shops reach for CDK because it's a first-party abstraction backed by a real programming language, with no separate templating DSL to learn. This lab builds the same category of infrastructure in CDK so the portfolio demonstrates real IaC breadth, not just one tool.

**Estimated time**: 60–75 minutes
**Cost**: ~$0–$1 (an S3 bucket and an SSM parameter cost fractions of a cent at this scale; the one-time `cdk bootstrap` creates a small S3 asset bucket that persists across projects — negligible ongoing cost, but not literally free forever if you never clean it up)

---

## Scenario
You've been asked to stand up a small piece of application infrastructure — a private, versioned S3 bucket for uploaded files, and an SSM parameter that another stack needs to reference — but this team deploys everything through CDK, not Terraform. You need to preview a change before every apply the same way `terraform plan` does, and you need to demonstrate CDK's actual differentiator over a templating language: real control flow, typed props, and one stack referencing another stack's resources without manually wiring up remote state. You also need to be ready to explain in an interview why you'd reach for CDK instead of Terraform on a given project — "because that's what the team uses" is true but not a complete answer.

---

## Objectives
- Install the CDK CLI and bootstrap an AWS account/region for CDK deployments
- Write CDK constructs in TypeScript: stacks, resources, props, and outputs
- Split a project into two stacks with a genuine cross-stack reference
- Run and interpret `cdk diff` before every `cdk deploy`
- Extract a reusable, typed `Construct` class — CDK's unit of reuse
- Reason through CDK vs. Terraform trade-offs well enough to defend a tool choice in an interview

---

## Part 1: Install CDK and Bootstrap the Account

### Step 1: Install the CDK CLI
```bash
npm install -g aws-cdk
cdk --version
```

### Step 2: Bootstrap the Account/Region
CDK needs a small set of supporting resources — an S3 bucket for deployment assets (Lambda zips, container image manifests, large templates) and IAM roles CloudFormation assumes during deploys — before it can deploy anything.

```bash
cdk bootstrap aws://<AWS_ACCOUNT_ID>/us-east-1
```

**What this does**: creates a `CDKToolkit` CloudFormation stack containing the asset bucket and a set of deployment roles. This is a **one-time operation per account/region**, not per project — every CDK app you build afterward in this account/region reuses it. It's the rough equivalent of Terraform's backend initialization, except CDK owns and manages it rather than you configuring a remote state backend yourself.

### Step 3: Scaffold the Project
```bash
mkdir cdk-aws-lab && cd cdk-aws-lab
cdk init app --language typescript
```
This generates a working TypeScript project — `bin/cdk-aws-lab.ts` (the app entry point), `lib/` (where stacks live), `package.json`, and `cdk.json` (CLI config). Unlike a Bicep or Terraform file, this is a real TypeScript program: it compiles, it can be unit-tested, and it can use loops, conditionals, and functions exactly like any other TypeScript code.

---

## Part 2: Define Two Stacks With a Cross-Stack Reference

### Step 4: The Data Stack
Replace the contents of `lib/cdk-aws-lab-stack.ts` — rename it conceptually to a `DataStack` holding the S3 bucket:

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as s3 from 'aws-cdk-lib/aws-s3';

export class DataStack extends cdk.Stack {
  public readonly bucket: s3.Bucket;

  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    this.bucket = new s3.Bucket(this, 'UploadsBucket', {
      versioned: true,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      encryption: s3.BucketEncryption.S3_MANAGED,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
      autoDeleteObjects: true, // lab-only: never do this on a real data bucket
    });

    new cdk.CfnOutput(this, 'BucketNameOutput', {
      value: this.bucket.bucketName,
    });
  }
}
```

**Reading this**: `this.bucket` is a real TypeScript class property, not a string identifier — the `AppConfigStack` in Step 5 references it as an object, not by parsing an ARN out of a lookup. `removalPolicy: DESTROY` and `autoDeleteObjects: true` exist purely so Cleanup can tear this down cleanly; CDK defaults to `RETAIN` for anything stateful specifically to stop `cdk destroy` from silently deleting production data.

### Step 5: The App Config Stack — Consuming the Data Stack's Output
Create `lib/app-config-stack.ts`:

```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as ssm from 'aws-cdk-lib/aws-ssm';
import * as s3 from 'aws-cdk-lib/aws-s3';

interface AppConfigStackProps extends cdk.StackProps {
  uploadsBucket: s3.Bucket;
}

export class AppConfigStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: AppConfigStackProps) {
    super(scope, id, props);

    new ssm.StringParameter(this, 'UploadsBucketNameParam', {
      parameterName: '/lab2/app/uploads-bucket-name',
      stringValue: props.uploadsBucket.bucketName,
    });
  }
}
```

**Why this matters**: passing `props.uploadsBucket` — the actual construct object from `DataStack` — is CDK doing something Bicep/ARM and raw CloudFormation can't do cleanly. Under the hood, CDK generates a CloudFormation `Export`/`Fn::ImportValue` pair between the two stacks automatically; you never hand-write the export name or worry about it drifting out of sync, because it's derived from a real object reference the TypeScript compiler checks at build time.

### Step 6: Wire Both Stacks in the App Entry Point
Edit `bin/cdk-aws-lab.ts`:

```typescript
#!/usr/bin/env node
import * as cdk from 'aws-cdk-lib';
import { DataStack } from '../lib/cdk-aws-lab-stack';
import { AppConfigStack } from '../lib/app-config-stack';

const app = new cdk.App();

const env = { account: '<AWS_ACCOUNT_ID>', region: 'us-east-1' };

const data = new DataStack(app, 'Lab2DataStack', { env });

new AppConfigStack(app, 'Lab2AppConfigStack', {
  env,
  uploadsBucket: data.bucket,
});
```

---

## Part 3: Synthesize, Diff, and Deploy

### Step 7: Synthesize to CloudFormation
```bash
cdk synth
```
**This is the core mechanic to understand**: CDK is not a separate deployment engine — it's a compiler. `cdk synth` runs your TypeScript, resolves every construct into CloudFormation resources, and writes real CloudFormation JSON/YAML to `cdk.out/`. Every CDK deployment is a CloudFormation deployment underneath; CDK's entire value proposition is generating that template from real code instead of you hand-writing it — the direct analog to Bicep compiling to ARM JSON.

**Validation checkpoint**: `cdk.out/Lab2DataStack.template.json` exists and contains an `AWS::S3::Bucket` resource with `VersioningConfiguration` and `PublicAccessBlockConfiguration` set as specified in Step 4.

### Step 8: Preview With cdk diff
```bash
cdk diff Lab2DataStack
cdk diff Lab2AppConfigStack
```
**What this does**: diffs the synthesized template against what's already deployed (nothing yet) and prints additions/changes/removals — CDK's equivalent of `terraform plan` or Bicep's `what-if`. On a fresh account this should show both stacks as entirely new resources with no in-place modifications.

**Validation checkpoint**: Confirm the diff shows only resource additions (`[+]`), nothing being replaced or destroyed, before proceeding.

### Step 9: Deploy Both Stacks
CDK resolves the dependency order automatically from the `uploadsBucket: data.bucket` reference — `Lab2DataStack` deploys before `Lab2AppConfigStack` without you specifying an explicit order:

```bash
cdk deploy --all --require-approval never
```
In a real pipeline, drop `--require-approval never` and let CDK prompt on any IAM/security-relevant change — this flag exists here only so the lab doesn't stall on an interactive prompt.

**Validation checkpoint**:
```bash
aws s3api get-bucket-versioning --bucket <bucket-name-from-CfnOutput>
aws ssm get-parameter --name /lab2/app/uploads-bucket-name --query Parameter.Value --output text
```
Expect `Status: Enabled` on versioning, and the SSM parameter's value matching the bucket name exactly — proof the cross-stack reference actually resolved to the real deployed resource, not a stale or hardcoded value.

---

## Part 4: A Reusable, Typed Construct

### Step 10: Extract a SecureBucket Construct
A raw `s3.Bucket` with the same five security properties repeated in every stack that needs a bucket is exactly the kind of duplication CDK's construct model exists to eliminate. Create `lib/constructs/secure-bucket.ts`:

```typescript
import { Construct } from 'constructs';
import * as cdk from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';

export interface SecureBucketProps {
  versioned?: boolean;
}

export class SecureBucket extends Construct {
  public readonly bucket: s3.Bucket;

  constructor(scope: Construct, id: string, props: SecureBucketProps = {}) {
    super(scope, id);

    this.bucket = new s3.Bucket(this, 'Bucket', {
      versioned: props.versioned ?? true,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      encryption: s3.BucketEncryption.S3_MANAGED,
      enforceSSL: true,
    });
  }
}
```

### Step 11: Use It From DataStack
Replace the inline `new s3.Bucket(...)` call in `lib/cdk-aws-lab-stack.ts` with:

```typescript
import { SecureBucket } from './constructs/secure-bucket';
// ...
const uploads = new SecureBucket(this, 'Uploads', { versioned: true });
this.bucket = uploads.bucket;
```

**Why this matters**: `SecureBucket` is now a typed, testable unit — every team member who needs "a bucket with our standard security baseline" imports this construct instead of copy-pasting five properties and hoping nobody forgets `blockPublicAccess`. This is the same idea as a Bicep module or Terraform module, but because it's a TypeScript class, it can also contain real logic — conditionals, loops over an array of bucket configs, or validation that throws a build-time error for a disallowed prop combination — not just declarative substitution.

**Validation checkpoint**: `cdk diff Lab2DataStack` shows `NoChange` (or purely cosmetic logical-ID differences if CDK regenerates a construct's ID) — proof the refactor produces an equivalent bucket, not a new one alongside the old.

---

## Part 5: Design Decision — CDK vs. Terraform

| Dimension | AWS CDK | Terraform |
|---|---|---|
| **Language model** | Real general-purpose language (TypeScript, Python, Java, C#, Go) — loops, conditionals, functions, and unit tests are native, not a DSL feature | HCL — declarative, purpose-built, deliberately not a general-purpose language; loops (`count`, `for_each`) and conditionals exist but are narrower |
| **State management** | No separate state file — CloudFormation tracks deployed resources server-side per stack | Explicit state file (local or remote backend); powerful but an operational artifact you must protect, lock, and share correctly |
| **Multi-cloud** | AWS-only (a variant, CDKTF, applies CDK's language model on top of Terraform providers, but plain CDK targets CloudFormation exclusively) | Multi-cloud (AWS, Azure, GCP, etc.) with one workflow and one state model across providers |
| **Type safety** | Compiler-checked — a wrong prop type or a typo'd property name fails at `tsc` build time, before `cdk synth` ever runs | Caught at `terraform plan`/`apply` time at the earliest; HCL has no compiler in the same sense |
| **Day-0 AWS feature support** | Depends on the CDK team authoring an L2 construct; raw CloudFormation resources (L1, `CfnXxx`) are available same-day, but the ergonomic L2 wrapper can lag | The `aws` provider needs a release cycle to catch up to brand-new AWS resource types, similar lag profile to CDK's L2 layer |
| **Team familiarity** | Natural fit for teams with strong software engineering backgrounds who want infrastructure defined the same way as application code | Natural fit for teams spanning multiple clouds, or already invested in HCL and the Terraform Registry's much larger module ecosystem |

**The loops/abstraction argument, concretely**: needing "one of these stacks per environment, with slightly different sizing" is a `for` loop over an array of config objects in CDK — no code duplication, no `count.index` indirection. Terraform does the same job with `for_each` over a map, but HCL's loop and conditional primitives are intentionally narrower than a real language's, which is exactly the trade Terraform makes for HCL staying simple, portable, and readable by someone who isn't a software engineer.

**When I'd pick each**: CDK, for an AWS-only environment with a team that wants infrastructure logic to be real, testable code — especially when the infrastructure has genuine conditional complexity (per-environment variations, generated resource sets) that would get awkward in HCL. Terraform, for anything multi-cloud, or where the team wants one declarative workflow and the breadth of the Terraform Registry's community modules across every provider, not just AWS.

---

## Cleanup

```bash
cdk destroy --all --force
```

**No local state file to clean up** — like Bicep's ARM deployment history, CloudFormation keeps stack records in AWS itself. `cdk destroy` deletes both stacks in reverse dependency order (`AppConfigStack` before `DataStack`, since the SSM parameter references the bucket).

The one-time `cdk bootstrap` stack (`CDKToolkit`) and its asset S3 bucket are reusable across every future CDK project in this account/region — **don't delete them** unless you're permanently done with CDK in this account, since removing them just means the next `cdk deploy` re-bootstraps automatically anyway.

---

## Key Concepts

| Term | Definition |
|---|---|
| **Construct** | CDK's fundamental unit of reuse and composition — anything from a single resource (L1) to a full opinionated pattern (L3) is a construct; `SecureBucket` in Part 4 is a custom one |
| **L1 vs. L2 vs. L3 constructs** | L1 (`CfnBucket`) maps 1:1 to a raw CloudFormation resource; L2 (`Bucket`) wraps it with sane defaults and typed props; L3 (patterns) composes multiple L2 constructs into a higher-level abstraction |
| **`cdk synth`** | Compiles the CDK app into CloudFormation templates — the CDK equivalent of `az bicep build`; every CDK deployment is a CloudFormation deployment underneath |
| **`cdk diff`** | Diffs the synthesized template against deployed stack state and prints additions/changes/removals without deploying — CDK's `terraform plan` / Bicep `what-if` equivalent |
| **`cdk bootstrap`** | One-time per account/region setup that creates the asset S3 bucket and IAM deployment roles every CDK app in that account/region reuses |
| **Cross-stack reference** | Passing a construct object (like `data.bucket`) between stacks; CDK auto-generates the underlying CloudFormation `Export`/`Fn::ImportValue` pair instead of you hand-wiring it |
| **`removalPolicy`** | Controls what happens to a stateful resource (bucket, table, database) when its stack is destroyed — CDK defaults to `RETAIN` specifically to prevent an accidental `cdk destroy` from deleting production data |

---

## Common Mistakes
- **Skipping `cdk diff` before `cdk deploy`**: the same mistake as running `terraform apply` without reading `plan` first — you find out what a deployment does by watching it happen instead of before
- **Setting `removalPolicy: DESTROY` and `autoDeleteObjects: true` out of habit**: correct for this disposable lab, a real production incident waiting to happen on an actual data store — CDK's `RETAIN` default is deliberate, don't override it without a specific reason
- **Hardcoding account ID/region inline everywhere instead of centralizing `env`**: makes the app harder to promote across accounts later; define `env` once in the app entry point (Step 6) and pass it down
- **Forgetting that L2 construct defaults still need review**: CDK's defaults are reasonable, not automatically compliant with every org's security baseline — always check what an L2 construct actually provisions by default (`cdk synth` and read the output) rather than assuming
- **Deleting the `CDKToolkit` bootstrap stack thinking it's project-specific cleanup**: it's shared infrastructure for every CDK app in that account/region — deleting it breaks other projects, not just this lab's

---

## Next Steps
Compare this against the Terraform approach to AWS infrastructure in [SAA-C03 Lab 8](../SAA-C03/aws-lab-8-terraform-aws.md) — same category of resource, different tooling philosophy. [Lab 3](lab-3-cicd-oidc.md) picks up from here and deploys this same CDK app through a GitHub Actions pipeline using OIDC federated auth instead of a stored credential. [Lab 7](lab-7-landing-zone-governance.md) revisits this CDK-vs-Terraform decision at the AWS Organizations governance layer, where the answer tips the other way.
