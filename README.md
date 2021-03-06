assert-aws-iam-permissions
===

[![Build Status](https://travis-ci.org/matt-deboer/assert-aws-iam-permissions.svg?branch=master)](https://travis-ci.org/matt-deboer/assert-aws-iam-permissions)
[![Docker Pulls](https://img.shields.io/docker/pulls/mattdeboer/assert-aws-iam-permissions.svg)](https://hub.docker.com/r/mattdeboer/assert-aws-iam-permissions/)

**assert-aws-iam-permissions** is a command-line utility for evaluation of AWS IAM policy documents against a set of asserted permissions (using the AWS Policy Simulation API).

Motivation
---

It was created specifically for use as an External Data Source in Terraform--used to assure that the expected permissions were actually enforced by a given policy before creating that policy.

Usage
---

```
NAME:
   assert-aws-iam-permissions

USAGE:
   assert-aws-iam-permissions [global options] command [command options] [arguments...]

VERSION:
   v0.6

COMMANDS:
     help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --policy-json value      The full contents of the IAM policy document; if empty,
                                assertions are read from JSON on stdin (under the key "policy_json") [$AAIP_POLICY_JSON]
   --max-length value       The maximum expected character length of the policy document (excluding whitespace);
                                a document greater than this length will cause an assertion failure (default: 0) [$AAIP_MAX_LENGTH]
   --assertions value       A JSON array of assertion statement objects, with the following structure:
                                  "comment":                  "This statement should be true",
                                  "expected_result":          "allowed|implicitDeny|explicitDeny|deny|denied" // 'deny' or 'denied' can be used to catch any deny type result
                                  "action_names":             ["service:Action"...],
                                  "resource_arns":            ["arn:aws:..."],
                                  "resource_policy":          "policy",
                                  "resource_owner":           "owner",
                                  "caller_arn":               "caller",
                                  "context_entries"": {
                                    "key": {"type": "the_type","values": ["some_values"...]},
                                    ...
                                  },
                                  "resource_handling_option": "option"
                                  if empty, assertions are read from JSON on stdin (under the key "assertions") [$AAIP_ASSERTIONS]
   --assume-role-arn value  The ARN of the role to assume when making AWS API calls [$AAIP_ASSUME_ROLE_ARN]
   --read-stdin, -i         whether to read inputs from stdin [$AAIP_READ_STDIN]
   --verbose, -V            Log debugging information [$AAIP_VERBOSE]
   --help, -h               show help
   --version, -v            print the version
```

Example Used in Terraform
---

This policy document example is trivial to evaluate, but it demonstrates configuration in terraform
for use in more complex policy scenarios.

**Note**: _this example assumes that `assert-aws-iam-permissions` is available on the path_

```hcl
data "aws_iam_policy_document" "my_policy" {
  statement {
    actions = [
      "s3:Get*",
    ]
    effect = "Allow"
    resources = ["*"]
  }
}

data "external" "validated_policy" {
  program = [ "assert-aws-iam-permissions", "--read-stdin" ]
  query = {
    policy_json = "${data.aws_iam_policy_document.my_policy.json}"
    max_length = 5120
    assertions = <<EOF
      [
        {
          "comment": "can read from a sub-path in 'my-bucket'",
          "expected_result": "allowed",
          "action_names": [
            "s3:GetObject"
          ],
          "resource_arns": [
            "arn:aws:s3:::my-bucket/some-sub-path"
          ]
        }
      ]
    EOF
  }
}

# we create the actual policy via the validated policy document
# policy creation will fail if it doesn't grant the asserted permissions
resource "aws_iam_policy" "my_policy" {
  name     = "my_policy"
  policy   = "${data.external.validated_policy.result["policy_json"]}"
}

```
