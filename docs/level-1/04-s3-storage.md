# 04 · S3 Storage

S3 (Simple Storage Service) is object storage: you store arbitrary files
("objects") inside named containers ("buckets"), addressed by key, with
effectively unlimited capacity and 11-nines durability. It's the backbone of
an enormous number of AWS architectures — static websites, data lakes,
backups, Lambda deployment packages, and the frontend of the capstone
project at the end of this level. This module covers buckets, objects,
versioning, bucket policies, and hosting a static website.

## Buckets and objects

A **bucket** is a globally-unique-named container; an **object** is a file
plus metadata, stored under a **key** (its full path-like name within the
bucket, e.g. `images/logo.png`).

```bash
# Bucket names must be globally unique across ALL of AWS, not just your account
aws s3 mb s3://yourname-training-bucket-2026 --region us-east-1

# Upload a file
echo "hello from s3" > hello.txt
aws s3 cp hello.txt s3://yourname-training-bucket-2026/hello.txt

# List objects in a bucket
aws s3 ls s3://yourname-training-bucket-2026/

# Download an object
aws s3 cp s3://yourname-training-bucket-2026/hello.txt downloaded.txt

# Sync a whole local directory to S3 (only uploads changed/new files)
aws s3 sync ./site/ s3://yourname-training-bucket-2026/site/

# Delete an object
aws s3 rm s3://yourname-training-bucket-2026/hello.txt
```

`aws s3` is a high-level, S3-specific command set (`cp`, `sync`, `mb`, `rb`,
`ls`, `rm`). There's also a lower-level `aws s3api` for operations like bucket
policies and versioning configuration that don't have a friendly `aws s3`
equivalent.

## Object versioning

**Versioning** keeps every version of an object instead of overwriting it —
protects against accidental deletes/overwrites, at the cost of storing every
version (which counts toward billing).

```bash
aws s3api put-bucket-versioning \
  --bucket yourname-training-bucket-2026 \
  --versioning-configuration Status=Enabled

# Overwrite the same key twice
echo "version 1" | aws s3 cp - s3://yourname-training-bucket-2026/notes.txt
echo "version 2" | aws s3 cp - s3://yourname-training-bucket-2026/notes.txt

# List every version that now exists for that key
aws s3api list-object-versions \
  --bucket yourname-training-bucket-2026 \
  --prefix notes.txt \
  --query "Versions[].[Key,VersionId,IsLatest]" --output table

# Fetch a specific (non-latest) version
aws s3api get-object \
  --bucket yourname-training-bucket-2026 \
  --key notes.txt \
  --version-id "<VersionId from above>" \
  restored-notes.txt
```

A "deleted" object in a versioned bucket isn't gone — S3 just adds a
**delete marker** on top; the prior versions are still retrievable until you
explicitly delete them by version ID.

## Bucket policies

A **bucket policy** is a resource-based policy (JSON, same shape as IAM
policies) attached directly to the bucket, controlling who can access it —
useful when you want to grant access without touching IAM at all (e.g.
public read for a static website).

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadForWebsite",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::yourname-training-bucket-2026/*"
    }
  ]
}
```

```bash
# S3 blocks public bucket policies by default (a safety net) — you must
# explicitly opt out of the relevant block before a public policy takes effect
aws s3api put-public-access-block \
  --bucket yourname-training-bucket-2026 \
  --public-access-block-configuration \
  BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false

aws s3api put-bucket-policy \
  --bucket yourname-training-bucket-2026 \
  --policy file://public-read-policy.json
```

!!! warning "Only make a bucket public on purpose"
    `RestrictPublicBuckets`/`BlockPublicPolicy` exist because accidentally
    public buckets are one of the most common real-world data leaks. Only
    disable the block for buckets that are genuinely meant to serve public
    content, like a static website's assets — never for buckets holding
    anything sensitive.

## Static website hosting

S3 can serve a bucket's contents directly as a website (no server needed) —
this is exactly the frontend pattern used in this level's capstone project.

```bash
aws s3api put-bucket-website \
  --bucket yourname-training-bucket-2026 \
  --website-configuration '{
    "IndexDocument": {"Suffix": "index.html"},
    "ErrorDocument": {"Key": "error.html"}
  }'

echo "<h1>Hello from S3 static hosting</h1>" > index.html
echo "<h1>404 - Not Found</h1>" > error.html
aws s3 cp index.html s3://yourname-training-bucket-2026/
aws s3 cp error.html s3://yourname-training-bucket-2026/

# The website endpoint URL follows a fixed pattern per region:
echo "http://yourname-training-bucket-2026.s3-website-us-east-1.amazonaws.com"
```

Visit that URL — with the public-read bucket policy from above in place,
your page loads with no web server, no EC2 instance, and no ongoing compute
cost.

## Storage classes (brief overview)

| Class | Use case | Relative cost |
|---|---|---|
| `STANDARD` | Frequently accessed data (default) | Baseline |
| `STANDARD_IA` | Infrequent access, still needs millisecond retrieval | Lower storage, retrieval fee |
| `GLACIER` | Long-term archive, retrieval takes minutes–hours | Much lower storage cost |

```bash
aws s3 cp bigfile.zip s3://yourname-training-bucket-2026/ --storage-class STANDARD_IA
```

For this course, `STANDARD` (the default) is fine — storage classes matter
more once you're optimizing costs at scale (Level 4 covers this).

## Cheat sheet

| Command | Purpose |
|---|---|
| `aws s3 mb s3://NAME` | Create a bucket. |
| `aws s3 cp FILE s3://BUCKET/KEY` | Upload/download an object. |
| `aws s3 sync DIR s3://BUCKET/PREFIX` | Sync a local directory to/from S3. |
| `aws s3 ls s3://BUCKET/` | List objects. |
| `aws s3 rm s3://BUCKET/KEY` | Delete an object. |
| `aws s3api put-bucket-versioning --bucket B --versioning-configuration Status=Enabled` | Turn on versioning. |
| `aws s3api list-object-versions --bucket B` | List all versions of all objects. |
| `aws s3api put-bucket-policy --bucket B --policy file://p.json` | Attach a bucket policy. |
| `aws s3api put-public-access-block ...` | Opt in/out of S3's public-access safety blocks. |
| `aws s3api put-bucket-website --bucket B --website-configuration '{...}'` | Enable static website hosting. |

## Exercise

1. Create a bucket with a globally-unique name of your choosing.
2. Enable versioning, upload the same key three times with different
   content, and use `list-object-versions` to view the history.
3. Enable static website hosting with a simple `index.html`, disable the
   relevant public-access blocks, and attach a bucket policy granting public
   `s3:GetObject`.
4. Visit the website endpoint URL in a browser and confirm your page loads.
5. Clean up: delete all object versions (not just the latest) and the bucket
   itself — `aws s3 rb s3://BUCKET --force` only removes current versions, so
   for a versioned bucket you may need to delete each version ID explicitly
   first via `aws s3api delete-object --version-id`.
