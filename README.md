# 🚀 Automating AWS with Pulumi: A Beginner's Guide to IaC

Hey there! I’m Veríssimo, a newbie in Infrastructure as Code (IaC) who got tired of setting up AWS instances by clicking through the console. It was slow, frustrating, and repetitive—but Pulumi changed everything! This guide is for you, someone like me, who wants to dive into IaC and automate AWS resources the easy way. I’ll share my journey—from an EC2 instance to a static S3 website with CloudFront—and highlight Pulumi’s amazing documentation that took me from zero to hero. Ready to join me?

---

## 📖 Introduction to IaC and Pulumi
![Demo](./assets/PulumiCode.png)

### What is IaC and Why Does It Matter?
IaC turns manual clicks (like in the AWS console) into code. It’s infrastructure defined in scripts: **replicable, automated, and human-error-proof**.

**Example:**  
Instead of manually creating 10 servers in AWS, a Pulumi script does it with one command.

### What’s Pulumi and How Is It Different?
Pulumi is an IaC tool that won me over because it uses **real programming languages** I already know, like Python and JavaScript—no tricky YAML or rigid HCL. I write infrastructure like I’m coding an app, and it works seamlessly with AWS, Azure, GCP, and more. The documentation at [www.pulumi.com/docs](https://www.pulumi.com/docs) is **fantastic**, guiding me step-by-step to create an EC2 instance and an S3 bucket without any headaches.

If you love coding and want to ditch the AWS console, Pulumi is your go-to. 🚀

---

## ⚠️ The Pain of Manually Creating Instances in AWS

Before Pulumi, I set up EC2 instances in the console:
- Clicking through endless options: instance type, networks, SSH keys...
- Repeating everything for each test, terrified of messing up.
- Wasting time I could’ve spent on cooler stuff.

This manual process killed my productivity. A simple test took 15 minutes, and I’d get lost in the console. IaC with Pulumi fixed that, and the docs showed me how.

---

## 🛠 Quick Setup: AWS CLI + Pulumi on Ubuntu

I set up my environment on Ubuntu (tip: on Windows, avoid accents in usernames!). Here’s the quick guide I followed:

### 1. Install AWS CLI
```bash
sudo apt update
sudo apt install unzip curl -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

### 2. Install Pulumi
```bash
curl -fsSL https://get.pulumi.com | sh
echo 'export PATH=$PATH:$HOME/.pulumi/bin' >> ~/.bashrc
source ~/.bashrc
pulumi version
```

### 3. Create an IAM User
In the AWS console:
1. Go to **IAM** > **User Groups** > **Create Group**.
2. Name: `PulumiUsers`.
3. Policies: `AmazonEC2FullAccess`, `AmazonS3FullAccess`, `CloudFrontFullAccess`, `IAMFullAccess`.
4. Create the group.
5. Go to **Users** > **Add users**.
6. Name: `pulumi-user`, access: **Programmatic access**, group: `PulumiUsers`.
7. Save the `Access Key ID` and `Secret Access Key` from the `.csv` file.
![Demo](./assets/userCredia.gif)

### 4. Configure AWS CLI
```bash
aws configure
```
- **Access Key ID** and **Secret Access Key**: From the IAM user.
- **Region**: `us-east-1`.
- **Output format**: `json`.
![AWS CLI](./assets/AwsCli.png)

Setup done!

---

## 🌟 My First Adventure: EC2 Instance

I kicked off with an EC2 instance using the `vm-aws-python` template:
```bash
pulumi new vm-aws-python
```
The documentation walked me through Python examples, and in minutes, I had a VM running. It was my first taste of IaC, and I was hooked! That success pushed me to tackle a static S3 site next.
![Architecture EC2](./assets/architecture.png)

---

## 🌍 Static S3 Website with CloudFront

### First Try: Windows 11
On Windows 11, my username "Veríssimo" (with an accent) messed up Pulumi’s paths. I tried switching to "Verissimo," but after rebooting, my password stopped working. I switched to Ubuntu, my dual-boot lifesaver.

### Success on Ubuntu
1. Created the project:
   ```bash
   mkdir my-site && cd my-site && pulumi new static-website-aws-python
   ```
   ![Structure](./assets/my-site.png)

2. Ran the deploy:
   ```bash
   pulumi up
   ```

### The Error I Hit
```
TypeError: BucketV2._internal_init() got an unexpected keyword argument 'website'
```
![Error](./assets/errorPulumi.png)

The original code used `website` in `BucketV2`, but the V2 API requires a separate setup.

### How I Fixed It
1. Basic bucket:
   ```python
   bucket = aws.s3.BucketV2("bucket")
   ```
2. Website config:
   ```python
   aws.s3.BucketWebsiteConfigurationV2(
       "bucket-website",
       bucket=bucket.bucket,
       index_document={"suffix": "index.html"},
       error_document={"key": "error.html"}
   )
   ```
3. Permissions:
   ```python
   aws.s3.BucketPublicAccessBlock("public-access-block", bucket=bucket.bucket, block_public_acls=False)
   aws.s3.BucketOwnershipControls("ownership-controls", bucket=bucket.bucket, rule={"object_ownership": "ObjectWriter"})
   ```
4. Deploy:
   ```bash
   pulumi up --refresh
   ```

### Final Code: `__main__.py`
I added CloudFront for better performance:
```python
import pulumi
import pulumi_aws as aws
import pulumi_synced_folder as synced_folder

config = pulumi.Config()
path = config.get("path") or "./www"
index_document = config.get("indexDocument") or "index.html"
error_document = config.get("errorDocument") or "error.html"

bucket = aws.s3.BucketV2("bucket")
bucket_website = aws.s3.BucketWebsiteConfigurationV2(
    "bucket-website", bucket=bucket.bucket,
    index_document={"suffix": index_document}, error_document={"key": error_document}
)
aws.s3.BucketOwnershipControls("ownership-controls", bucket=bucket.bucket, rule={"object_ownership": "ObjectWriter"})
aws.s3.BucketPublicAccessBlock("public-access-block", bucket=bucket.bucket, block_public_acls=False)
bucket_folder = synced_folder.S3BucketFolder(
    "bucket-folder", acl="public-read", bucket_name=bucket.bucket, path=path,
    opts=pulumi.ResourceOptions(depends_on=[ownership_controls, public_access_block])
)

cdn = aws.cloudfront.Distribution(
    "cdn", enabled=True,
    origins=[{
        "origin_id": bucket.arn, "domain_name": bucket_website.website_endpoint,
        "custom_origin_config": {"origin_protocol_policy": "http-only", "http_port": 80, "https_port": 443, "origin_ssl_protocols": ["TLSv1.2"]}
    }],
    default_cache_behavior={
        "target_origin_id": bucket.arn, "viewer_protocol_policy": "redirect-to-https",
        "allowed_methods": ["GET", "HEAD", "OPTIONS"], "cached_methods": ["GET", "HEAD", "OPTIONS"],
        "default_ttl": 600, "max_ttl": 600, "min_ttl": 600,
        "forwarded_values": {"query_string": True, "cookies": {"forward": "all"}}
    },
    price_class="PriceClass_100",
    custom_error_responses=[{"error_code": 404, "response_code": 404, "response_page_path": f"/{error_document}"}],
    restrictions={"geo_restriction": {"restriction_type": "none"}},
    viewer_certificate={"cloudfront_default_certificate": True}
)

pulumi.export("originURL", pulumi.Output.concat("http://", bucket_website.website_endpoint))
pulumi.export("cdnURL", pulumi.Output.concat("https://", cdn.domain_name))
```

> 💡 **Tip:** Run `pulumi up ` 

![output](./assets/output1.png)


**What it does:**
- Sets up an S3 bucket as a static website.
- Syncs the local `./www` folder to the bucket.
- Adds CloudFront for fast, secure delivery (HTTPS).

---

## 📚 Lessons Learned
- **Windows:** Avoid accents in usernames.
- **S3 V2:** Website setup is separate now.
- **Pulumi:** Python code and great docs are a winning combo.
- **CloudFront:** Boosts speed and security effortlessly.


---

## 🖼 Final Architecture
![S3](./assets/architectureS3.png)

---

## 🎉 Check It Out Live
After deploying, here’s where you can see my work in action:
- **Diary:** [https://d33ejg1jsmvn6g.cloudfront.net/index.html](https://d33ejg1jsmvn6g.cloudfront.net/index.html) – My personal story with Pulumi.
- **Tutorial:** [https://d33ejg1jsmvn6g.cloudfront.net/tutorial.html](https://d33ejg1jsmvn6g.cloudfront.net/tutorial.html) – The step-by-step guide to replicate it.


---
I hope my journey inspires you to explore IaC with Pulumi! Got questions? Let’s chat!
