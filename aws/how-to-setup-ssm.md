# How to connect to AWS EC2 instance using AWS CLI with Session Manager

1. Install AWS SSM agent on the EC2 instance if it doesn't be installed on your instance yet. Please refer the [official document](https://docs.aws.amazon.com/zh_tw/systems-manager/latest/userguide/sysman-manual-agent-install.html).
1. Install AWS CLI on your local workstation. Please refer the [official document](https://docs.aws.amazon.com/zh_tw/cli/latest/userguide/install-cliv2-windows.html) for Windows.
1. Install Session Manager Plugin on your local workstation. Please refer the [official document](https://docs.aws.amazon.com/zh_tw/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html#install-plugin-windows) for Windows.
1. Create a minimum policy in IAM to allow Session Manager execution with following content:

    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "ssm:UpdateInstanceInformation",
                    "ssmmessages:CreateControlChannel",
                    "ssmmessages:CreateDataChannel",
                    "ssmmessages:OpenControlChannel",
                    "ssmmessages:OpenDataChannel"
                ],
                "Resource": "*"
            },
            {
                "Effect": "Allow",
                "Action": [
                    "s3:GetEncryptionConfiguration"
                ],
                "Resource": "*"
            }
        ]
    }
    ```

    Please refer the [official document](https://docs.aws.amazon.com/zh_tw/systems-manager/latest/userguide/getting-started-create-iam-instance-profile.html).

1. Select your IAM role which your EC2 instance uses, and attach the above policy on the role.
1. Create or select an existing IAM user and add an inline policy with following content:

    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "ssm:StartSession",
                    "ssm:SendCommand"
                ],
                "Resource": [
                    "arn:aws:ec2:<region>:<account-id>:instance/your-instance-id",
                    "arn:aws:ssm:<region>:<account-id>:document/SSM-SessionManagerRunShell"
                ]
            },
            {
                "Effect": "Allow",
                "Action": [
                    "ssm:DescribeSessions",
                    "ssm:GetConnectionStatus",
                    "ssm:DescribeInstanceInformation",
                    "ssm:DescribeInstanceProperties",
                    "ec2:DescribeInstances"
                ],
                "Resource": "*"
            },
            {
                "Effect": "Allow",
                "Action": [
                    "ssm:TerminateSession"
                ],
                "Resource": [
                    "arn:aws:ssm:*:*:session/${aws:username}-*"
                ]
            }
        ]
    }
    ```

    This policy will enable the connection using Session Manager console, CLI and Amazon EC2 console. Please refer the [official document](https://docs.aws.amazon.com/zh_tw/systems-manager/latest/userguide/getting-started-restrict-access-quickstart.html).

1. (Optional) Create an access key in your IAM user if you don't have one or you want to use an new key.
1. In your local workstation, create an new profile with following command in order to connect to you EC2 instance: `aws configure --profile your-profile-name`.
1. Connect to your EC2 instance using AWS CLI command: `aws ssm start-session --target your-instance-id --profile your-profile-name`.
