---
title: Securely configuring your AWS account
categories:
  - Security
tags:
  - AWS
  - Cloud
---

Recently I decided to sit down and futher lock down my personal AWS account. I haven't used it for much other than S3 storage of macOS installers and in turn had not configured things as securely as I would have liked. The following post walks you through how to lock down an AWS account that is used by a single user. A lot of the recommendations apply just as much to an account with multiple users as well.

First just a quick note. Amazon has a lot of documentation for AWS. Everything I cover here can be found in their guides and documentation but you have to wade through a lot of different documents. My goal here is to provide a short easy to follow guide. The two following documents are foundational to what I cover and would be useful to read over.

[AWS General Reference](https://docs.aws.amazon.com/general/latest){: target="_blank"}  
[AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide){: target="_blank"}

# AWS root user

When you first sign up for AWS you start with a single root user account that has complete access to all AWS services and resources. This root user account should only be used for setting up your first IAM user or the small handful of tasks that require root user access. With that said the following steps should be taken to lock down access to the root user account:

1. Ensure the root user has a [strong password set](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_passwords_change-root.html){: target="_blank"}.
2. [Enable Multi-factor authentication (MFA)](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-user.html#id_root-user_manage_mfa){: target="_blank"} for the root user.
3. [Delete any access keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-user.html#id_root-user_manage_delete-key){: target="_blank"} you might have for the root user.

# IAM Policies

[IAM policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html){: target="_blank"} allow you to define permissions. In order to grant permissions to users or services you need to first create some policies. Alternatively there are a lot of standard policies that are provided in AWS already that you can use. For example there is an `AdministratorAccess` policy that provides full access to all AWS services and resources. You use policies by attaching them to IAM identities (users, groups or roles). We're going to [create](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_create-console.html){: target="_blank"} three different policies for our setup.

## IAMRequireMFA

The first policy, `IAMRequireMFA`, will be used to allow all IAM users the ability to set set a password and configure MFA on their IAM user account. It will ensure that MFA is required for any other action the user wants to take.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ListMiscAccountInformation",
            "Effect": "Allow",
            "Action": [
                "iam:GetAccountPasswordPolicy",
                "iam:GetAccountSummary",
                "iam:ListAccountAliases"
            ],
            "Resource": [
                "*"
            ]
        },
        {
            "Sid": "AllowAllUsersToListUserAccounts",
            "Effect": "Allow",
            "Action": [
                "iam:ListUsers"
            ],
            "Resource": [
                "arn:aws:iam::*:user/"
            ]
        },
        {
            "Sid": "AllowIndividualUserToChangeTheirPassword",
            "Effect": "Allow",
            "Action": [
                "iam:ChangePassword"
            ],
            "Resource": [
                "arn:aws:iam::*:user/${aws:username}"
            ]
        },
        {
            "Sid": "AllowIndividualUserToListTheirMFA",
            "Effect": "Allow",
            "Action": [
                "iam:ListVirtualMFADevices",
                "iam:ListMFADevices"
            ],
            "Resource": [
                "arn:aws:iam::*:mfa/",
                "arn:aws:iam::*:user/${aws:username}"
            ]
        },
        {
            "Sid": "AllowIndividualUserToManageTheirMFA",
            "Effect": "Allow",
            "Action": [
                "iam:CreateVirtualMFADevice",
                "iam:DeleteVirtualMFADevice",
                "iam:EnableMFADevice",
                "iam:ResyncMFADevice"
            ],
            "Resource": [
                "arn:aws:iam::*:mfa/${aws:username}",
                "arn:aws:iam::*:user/${aws:username}"
            ]
        },
        {
            "Sid": "RequireMFAForUserToManageRemainderOfAccount",
            "Effect": "Allow",
            "Action": [
                "iam:CreateLoginProfile",
                "iam:DeleteLoginProfile",
                "iam:GetLoginProfile",
                "iam:UpdateLoginProfile"
            ],
            "Resource": [
                "arn:aws:iam::*:user/${aws:username}"
            ],
            "Condition": {
                "Bool": {
                    "aws:MultiFactorAuthPresent": true
                }
            }
        },
        {
            "Sid": "RequireMFAToDeactivateTheirOwnVirtualMFADevice",
            "Effect": "Allow",
            "Action": [
                "iam:DeactivateMFADevice"
            ],
            "Resource": [
                "arn:aws:iam::*:mfa/${aws:username}",
                "arn:aws:iam::*:user/${aws:username}"
            ],
            "Condition": {
                "Bool": {
                    "aws:MultiFactorAuthPresent": true
                }
            }
        },
        {
            "Sid": "DoNotAllowAnythingOtherThanAboveUnlessMFA",
            "Effect": "Deny",
            "NotAction": "iam:*",
            "Resource": "*",
            "Condition": {
                "Null": {
                    "aws:MultiFactorAuthAge": "true"
                }
            }
        }
    ]
}
```

## AssumeAdminRole

The second policy, `AssumeAdminRole`, will only be used by users that you want to grant admin privileges to. We haven't discussed IAM roles yet so just make note that this policy grants permission to assume a role named `AdminRole`.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": [
                "arn:aws:iam::*:role/AdminRole"
            ],
            "Condition": {
                "Bool": {
                    "aws:MultiFactorAuthPresent": true,
                    "aws:SecureTransport": true
                },
                "NumericLessThan": {
                    "aws:MultiFactorAuthAge": "3600"
                }
            }
        }
    ]
}
```

## AdminPolicy

The final policy, `AdminPolicy`, we define will be used to specify admin privleges. This policy will be specific to your needs but for now you can set up your policy exactly the same as the built in AWS `AdministratorAccess` policy. Please take care though, this policy grants access to everything in the AWS account.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "*",
            "Resource": "*"
        }
    ]
}
```

# IAM Roles

An [IAM role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html){: target="_blank"} is an identity that you can attach IAM policies to. You can use roles to delegate access to different AWS services. An IAM user can log in and then [switch to a different role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-console.html){: target="_blank"} to perform the actions they need. This is called "assuming a role". The advantage of assuming a role is that when do, a set of temporary short lived credentials are transparently issued to your user and those are used to perform the action. This means if the credentials were ever somehow compromised they only are useful for an hour or two. In our case we're going to [create a single role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-user.html){: target="_blank"}.

## AdminRole

The `AdminRole` will have the `AdminPolicy` attached to it. If you're creating the role from the AWS console simply choose the `Another AWS account` option but use your own account ID when setting it up. You can check the `Require MFA` checkbox if you want but our policies above should also ensure that MFA is required in order to assume the role.

# IAM Groups

IAM groups are another type of identity that policies can be attached to. IAM users can be added to groups and then a single policy can apply to all users of the group at once. We're going to [create](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_groups_create.html){: target="_blank"} two separate groups.

## Users

Any new IAM user should be added to this group. The permissions you attach here are meant to apply to all users. In our case we're going to attach the `IAMRequireMFA` policy to this group.

## Administrators

Only certain users should be granted administrative access to your account. The `AdminPolicy` should be attached to this group.

# IAM Users

With our policies, roles and groups set up we can now finally [create a new IAM user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html){: target="_blank"}. You can give your user whatever user name you want. For our usage go ahead and check the `Programatic access` and `AWS Management Console access` check boxes. Add the user to both the `Users` group and the `Administrators` group. Click through and finish creating the user. When prompted make sure to save your newly created access keys somewhere secure.

# AWS CLI

Finally we can [install](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html){: target="_blank"} and [configure](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html){: target="_blank"} the AWS command line interface. The Amazon docs do a good job of covering all this so I'll just cover how you set things up to use the `AdminRole` we created.

## ~/.aws/credentials

Add the access keys you saved after creating your IAM user into the `credentials` file.

```
[default]
aws_access_key_id=AKIAIOSFODNN7EXAMPLE
aws_secret_access_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

## ~/.aws/profile

In the `profile` we're going to [configure the IAM role](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-role.html){: target="_blank"} we created earlier.

```
[profile admin]
role_arn = arn:aws:iam::123456789012:role/AdminRole
source_profile = default
mfa_serial = arn:aws:iam::123456789012:mfa/username
```

Make sure to change `123456789012` and `username` to your account ID and the user name you created earlier.

# Testing it out

Finally we can test things out. Running an `aws` command will use our default access keys and only have access to assume the `AdminRole`. If we try to list IAM groups we'll get an error.

```sh
$ aws iam list-groups

An error occurred (AccessDenied) when calling the ListGroups operation: User: arn:aws:iam::123456789012:user/username is not authorized to perform: iam:ListGroups on resource: arn:aws:iam::123456789012:group/
```

If we instead add the `--profile admin` argument to our command we'll be asked to authorize with our configured MFA and then the command can run.

```sh
$ aws iam list-groups --profile admin
Enter MFA code for arn:aws:iam::123456789012:mfa/username:
{
    "Groups": [
        {
            "Path": "/",
            "GroupName": "Administrators",
            "GroupId": "ABPAQDWXIEOQBGZSQSWBN",
            "Arn": "arn:aws:iam::123456789012:group/Administrators",
            "CreateDate": "2020-09-09T14:38:41Z"
        },
        {
            "Path": "/",
            "GroupName": "Users",
            "GroupId": "ABPAQDWXIEOQLAFJFH43Y",
            "Arn": "arn:aws:iam::123456789012:group/Users",
            "CreateDate": "2020-09-09T14:20:32Z"
        }
    ]
}
```

# Conclusion

It does take a bit of work to set all of this up but ultimately I think it provides a good set up for a personal account. As a review we've done the following.

* Locked down the root user with MFA and no access keys.
* Created an IAM user who only has permission to do the following:
  * Set a password
  * Configure MFA
  * Assume the role we setup
* Created a role with admin permissions.

This set up ensures that when making use of our admin permissions we assume the role and need to enter a MFA token in order for it to work. The assumed role uses transparent temporary credentials that are only good for one hour. If our IAM users access keys are compromised we've made sure they don't have permission to do any harm and the attacker would need our MFA device in order to assume the admin role. I think this set up strikes a good balance between security and usability.
