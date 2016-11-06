# ami2docker

A tiny CLI tool to convert Amazon [EC2 AMIs](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html) to [Docker images](https://docs.docker.com/engine/tutorials/dockerimages/). Useful for faster local debugging, emulating a [Lambda](http://docs.aws.amazon.com/lambda/latest/dg/welcome.html)-esque environment, etc.

## Usage

```
$ ami2docker --help
Usage:
    ami2docker [OPTIONS]

Options:
    --ami-name AMI NAME           Name of AMI to turn into Docker image
    --ssm-bucket S3 BUCKET        Bucket for SSM output
    --subnet-id SUBNET ID         Subnet to launch temporary instance into
    --instance-profile INSTANCE PROFILE Name of instance profile
    --instance-type INSTANCE TYPE Instance type (default: "m4.large")
    --instance-ami-name INSTANCE AMI NAME AMI to use for temporary instance (default: "amzn-ami-hvm-2016.03.3.x86_64-gp2")
    --ecr-registry ECR REGISTRY   Registry for Docker image to be uploaded to
    -h, --help                    print help

```

* `--ami-name` is mandatory. It is the name of the AMI that you wish to turn into a Docker image. For example, `amzn-ami-hvm-2016.03.3.x86_64-gp2` is the name of a recent-ish Amazon Linux AMI.
* `--ssm-bucket` is mandatory. It is where output from commands executed on a temporary EC2 instance are saved to. Commands are executed by means of the AWS [SSM](http://docs.aws.amazon.com/ssm/latest/APIReference/Welcome.html) `SendCommand` API, so connectivity from your machine to the temporary instance is not required. The instance _does_ need connectivity to the Internet, which is currently achieved by requesting a public IP.
* `--subnet-id` is optional. If not supplied, the instance will boot into the default subnet. If this is unsuitable, an alternative subnet ID can be provided. Note that the subnet requires connectivity to AWS SSM, either by means of an Internet gateway, NAT instance, VPN, etc.
* `--instance-profile` is optional. It is the name an of EC2 [instance profile](http://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html) used to grant access to the temporary instance to interact with SSM and upload the Docker image to the [ECR](https://aws.amazon.com/ecr/) registry. If not provided, an IAM role with sufficient permissions and an instance profile will be created on your behalf.
* `--instance-type` is optional. It defaults to `m4.large` but can be a smaller instance if you wish to save pennies.
* `--instance-ami-name` is the AMI to use to boot the temporary instance. It defaults to a recent Amazon Linux AMI. It does not need to match the AMI being dockerised nor does it likely need to be changed.
* `--ecr-registry` is the ECR registry to upload to. It should be in the form ` <account number>.dkr.ecr.<region>.amazonaws.com/<repo name>`. If not provided, a registry will be created on your behalf in the current region with the AMI name as the repo name.

## How it works

* The details for the provided AMI name are looked up. 
* A new EBS volume is created from the AMI's root volume snapshot.
* A temporary EC2 instance is spun up.
* The new volume is attached as a secondary volume to this instance and mounted at `/mnt`.
* The volume at `/mnt` is piped into `docker import`.
* The newly-created image is uploaded to AWS ECR.
* Temporary Instance is terminated.

## Required IAM permissions

The following permissions are required by the role (user) invoking `ami2docker`:

```json
{
  "PolicyName": "root",
  "PolicyDocument": {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "Stmt1478418071533",
        "Action": [
          "iam:AttachRolePolicy",
          "iam:CreateInstanceProfile",
          "iam:CreateRole",
          "iam:ListRoles",
          "iam:PassRole"
        ],
        "Effect": "Allow",
        "Resource": "*"
      },
      {
        "Sid": "Stmt1478418169890",
        "Action": [
          "ec2:AttachVolume",
          "ec2:CreateVolume",
          "ec2:DescribeImages",
          "ec2:DescribeInstances",
          "ec2:DescribeVolumes",
          "ec2:RunInstances"
        ],
        "Effect": "Allow",
        "Resource": "*"
      },
      {
        "Sid": "Stmt1478418198770",
        "Action": [
          "ssm:DescribeInstanceInformation",
          "ssm:ListCommandInvocations",
          "ssm:SendCommand"
        ],
        "Effect": "Allow",
        "Resource": "*"
      },
      {
        "Sid": "Stmt1478418219486",
        "Action": [
          "ecr:CreateRepository"
        ],
        "Effect": "Allow",
        "Resource": "*"
      },
      {
        "Sid": "Stmt1478418237337",
        "Action": [
          "s3:CreateBucket",
          "s3:GetObject"
        ],
        "Effect": "Allow",
        "Resource": "*"
      }
    ]
  }
}
```

## TODO

* Document which permissions aren't needed when all parameters are specified
* Better error handling
* Pretty, informative logging
