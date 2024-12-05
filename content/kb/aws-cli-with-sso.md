---
title: "AWS CLI with Okta SSO"
categories: ['kb']
tags: ['aws', 'cli', 'okta', 'sso']
---

# Summary

This page explains how to authenticate AWS CLI with Okta SSO. These instructions
assume the use of fish shell.

# Pre-requisites

Have these tools installed before proceeding:

  * [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
  * [saml2aws](https://github.com/Versent/saml2aws)

# How to

Configure an IDP account under the alias `my-account`:

```bash
$ saml2aws configure \
    --idp-account my-account \
    --idp-provider Okta \
    --url https://example.okta.com/home/amazon_aws/0a1b2c/000 \
    --username me@example.com \
    --role arn:aws:iam::1234567890:role/my-role \
    --region eu-central-1
```

The command above will create a new entry in the file `~/.saml2aws`. The file
should look like this:

```bash
$ cat ~/.saml2aws
[my-account]
name                    = my-account
app_id                  =
url                     = https://example.okta.com/home/amazon_aws/0a1b2c/000
username                = me@example.com
provider                = Okta
mfa                     = PUSH
mfa_ip_address          =
skip_verify             = false
timeout                 = 0
aws_urn                 = urn:amazon:webservices
aws_session_duration    = 3600
aws_profile             = my-account
resource_id             =
subdomain               =
role_arn                = arn:aws:iam::1234567890:role/my-role
region                  = eu-central-1
http_attempts_count     =
http_retry_delay        =
credentials_file        =
saml_cache              = false
saml_cache_file         =
target_url              =
disable_remember_device = false
disable_sessions        = false
download_browser_driver = false
headless                = false
prompter                =
```

To login to `my-account`, run:

```bash
$ saml2aws login -a my-account
```

After entering your password, you should be authenticated.

Use the following command to export the AWS credentials to environment variables: 

```bash
$ saml2aws script --idp-account=my-account --shell=fish | source
```

Create an alias to make the process of logging in and exporting variables easier:

```bash
$ function s2a
      saml2aws login -a $argv
      saml2aws script --idp-account=$argv --shell=fish | source
  end
$ funcsave s2a
```

To use the newly created alias:

```bash
$ s2a my-account
```
