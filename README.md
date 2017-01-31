# rabblerouser-infrastructure

Code for spinning up Rabble Rouser infrastructure and provisioning the app.

## First-time setup

Right now this is not suitable for end users to be running on their own. You probably need to be at least semi-technical
for these instructions to make sense.

1. [Install Terraform 0.8.x](https://www.terraform.io/intro/getting-started/install.html)
2. [Install the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/installing.html)
3. [Create an AWS account](https://aws.amazon.com/)
4. [Create an AWS API key pair](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html)
5. Put your credentials in `~/.aws/credentials` like this:

    ```
    [default]
    aws_access_key_id=AKIAEXAMPLEEXAMPLE
    aws_secret_access_key=abc123abc123abc123+abc123abc123
    ```

6. Create a Route53 hosted zone for the parent domain of where your Rabble Rouser will live. E.g., if you are going to
 deploy it at rabblerouser.example.com, then you'll need to have a Route53 hosted zone already created for example.com.
7. Create an S3 bucket to store the [terraform state file](https://www.terraform.io/docs/state/) in (see notes below):

    ```sh
    aws s3api create-bucket --bucket my-rabblerouser-tf-state --region us-east-1 --create-bucket-configuration LocationConstraint=us-east-1 --acl private
    ```

8. Configure terraform to use the bucket:

    ```sh
    terraform remote config \
      -backend=s3 \
      -backend-config="bucket=my-rabblerouser-tf-state" \
      -backend-config="key=terraform.tfstate" \
      -backend-config="region=us-east-1"
    ```

This will create a local file `.terraform/terraform.tfstate`, to remember where your state is stored. You can ignore
this file, and if you lose it, you can regenerate it with the above command.

Notes on the S3 bucket:
 - The bucket name must be unique to the whole world (not just your AWS account)
 - Terraform will store all of its state in the bucket, some of which is sensitive, so don't share access with anyone who doesn't need it
 - Terraform won't manage the bucket itself, just its contents

## Create all your infrastructure

1. Create a file `terraform.tfvars` to store your project-specific config in locally, with contents similar to the below.
 See `variables.tf` for descriptions of what these variables are.

    ```tfvars
    # This file will contain secrets, so never check it in to source control, and never share it with untrusted people!

    # These variables have no default values and must be supplied
    route53_zone_id = "ABC123"
    domain = "rabblerouser.example.com"
    tls_cert_email = "webmaster@rabblerouser.team"
    session_secret = "something long and random without special chars"
    db_password = "something long and random without special chars"

    # These lines may be removed if you're happy with the defaults shown here
    app_git_sha = "HEAD"
    private_key_path = "~/.ssh/id_rsa"
    public_key_path = "~/.ssh/id_rsa.pub"
    region = "us-east-1"
    ```

2. Do a dry run first to see what terraform will do:

    ```
    terraform plan
    ```

3. Create the infrastructure: *Warning: this will cost you money, even if you have the AWS free tier!*

    ```
    terraform apply
    ```

## Re-run the provisioner

You might want to do this because the ansible code has changed, or to deploy the latest version of the app:

```sh
terraform taint null_resource.provisioner
terraform apply
```

## Destroy all your infrastructure

Warning: this... well it will destroy all your infrastructure!

```sh
terraform destroy
```

## Running it without a tfvars config file

If using a file is inconvenient (e.g. when running in a CI pipeline), you can use CLI flags or environment variables
instead (see the [terraform docs](https://www.terraform.io/intro/getting-started/variables.html) for more detail):

```sh
terraform apply -var 'route53_zone_id="ABC123"'
# OR
TF_VAR_route53_zone_id="ABC123" terraform apply
```
