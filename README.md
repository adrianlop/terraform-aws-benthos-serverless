# terraform-aws-benthos-serverless
a Terraform module for deploying [Benthos](https://www.benthos.dev/) as an AWS Lambda Function with support for [Gomplate](https://docs.gomplate.ca/) templated config

## Getting Started
```terraform
# store sensitive configuration in ssm parameter store
resource "aws_ssm_parameter" "secrets" {
  for_each = {
    "slack-channel" = var.slack_channel
    "slack-token"   = var.slack_token
  }
  name  = "/benthos-lambda-example/${each.key}"
  type  = "SecureString"
  value = each.value
}

# deploy benthos lambda function using ssm as config datasource
module "benthos_lambda" {
  source = "cludden/aws/benthos-serverless"

  name = "benthos-lambda-example"

  config_datasources = {
    ssm = "aws+smp:///benthos-lambda-example"
  }

  config = <<-YAML
    pipeline:
      processors:
      # format slack chat.postMessage payload
      - bloblang: |
          channel = "{{ (ds "ssm" "slack-channel").Value }}"
          text = "EC2 Instance %s has been terminated.".format(detail.EC2InstanceId)
          blocks = [{
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "EC2 Instance `%s` has been terminated.".format(detail.EC2InstanceId)
            }
          }, {
            "type": "context",
            "elements": [{
              "type": "mrkdwn",
              "text": "*account:* %s".format(account)
            }, {
              "type": "mrkdwn",
              "text": "*region:* %s".format(region)
            }, {
              "type": "mrkdwn",
              "text": "*group:* %s".format(detail.AutoScalingGroupName)
            }]
          }]

    output:
      switch:
        retry_until_success: false
        cases:
        - check: errored()
          output:
            reject: "$${!error()}"
        - output:
            http_client:
              url: https://slack.com/api/chat.postMessage
              propagate_response: true
              headers:
                Authorization: Bearer {{ (ds "ssm" "slack-token").Value }}
                Content-Type: application/json; charset=utf-8
  YAML
}
```

<!-- BEGIN_TF_DOCS -->
## Requirements

| Name | Version |
|------|---------|
| <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) | >= 1.0.0 |
| <a name="requirement_aws"></a> [aws](#requirement\_aws) | >= 3.0.0 |
| <a name="requirement_get"></a> [get](#requirement\_get) | 0.1.0 |

## Providers

| Name | Version |
|------|---------|
| <a name="provider_aws"></a> [aws](#provider\_aws) | >= 3.0.0 |
| <a name="provider_get"></a> [get](#provider\_get) | 0.1.0 |

## Modules

No modules.

## Resources

| Name | Type |
|------|------|
| [aws_cloudwatch_log_group.this](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_log_group) | resource |
| [aws_iam_role.this](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role) | resource |
| [aws_lambda_function.this](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lambda_function) | resource |
| [aws_lambda_layer_version.gomplate](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lambda_layer_version) | resource |
| [get_artifact.benthos](https://registry.terraform.io/providers/cludden/get/0.1.0/docs/resources/artifact) | resource |
| [get_artifact.gomplate](https://registry.terraform.io/providers/cludden/get/0.1.0/docs/resources/artifact) | resource |
| [aws_iam_policy_document.this](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/iam_policy_document) | data source |
| [aws_iam_policy_document.trust](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/iam_policy_document) | data source |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_benthos_version"></a> [benthos\_version](#input\_benthos\_version) | benthos artifact version | `string` | `"3.62.0"` | no |
| <a name="input_config"></a> [config](#input\_config) | gomplate templated benthos config (YAML format) | `string` | n/a | yes |
| <a name="input_config_datasources"></a> [config\_datasources](#input\_config\_datasources) | map of gomplate datasources referenced by config | `map(string)` | `{}` | no |
| <a name="input_description"></a> [description](#input\_description) | lambda function description | `string` | `"benthos-lambda"` | no |
| <a name="input_environment"></a> [environment](#input\_environment) | map of additional environment variables | `map(string)` | `{}` | no |
| <a name="input_gomplate_version"></a> [gomplate\_version](#input\_gomplate\_version) | gomplate-lambda-extension artifact version | `string` | `"0.1.1"` | no |
| <a name="input_memory_size"></a> [memory\_size](#input\_memory\_size) | amount of memory in MB your Lambda Function can use at runtime | `number` | `128` | no |
| <a name="input_name"></a> [name](#input\_name) | function name | `string` | n/a | yes |
| <a name="input_reserved_concurrent_executions"></a> [reserved\_concurrent\_executions](#input\_reserved\_concurrent\_executions) | amount of reserved concurrent executions for this lambda function | `number` | `-1` | no |
| <a name="input_retention_in_days"></a> [retention\_in\_days](#input\_retention\_in\_days) | function log retention in days | `number` | `7` | no |
| <a name="input_role_arn"></a> [role\_arn](#input\_role\_arn) | execution role arn | `string` | `null` | no |
| <a name="input_role_name"></a> [role\_name](#input\_role\_name) | override default execution role name | `string` | `null` | no |
| <a name="input_security_group_ids"></a> [security\_group\_ids](#input\_security\_group\_ids) | list of vpc security group ids | `list(string)` | `[]` | no |
| <a name="input_statements"></a> [statements](#input\_statements) | customize role policy statements | <pre>list(object({<br>    actions = list(string)<br>    conditions = optional(list(object({<br>      test     = string<br>      variable = string<br>      values   = list(string)<br>    })))<br>    effect    = optional(string)<br>    resources = list(string)<br>  }))</pre> | `[]` | no |
| <a name="input_subnet_ids"></a> [subnet\_ids](#input\_subnet\_ids) | list of vpc subnet ids | `list(string)` | `[]` | no |
| <a name="input_timeout"></a> [timeout](#input\_timeout) | function timeout in seconds | `number` | `3` | no |

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_arn"></a> [arn](#output\_arn) | function arn |
| <a name="output_id"></a> [id](#output\_id) | function name |
| <a name="output_role_arn"></a> [role\_arn](#output\_role\_arn) | function role arn |
| <a name="output_role_id"></a> [role\_id](#output\_role\_id) | function role name |
<!-- END_TF_DOCS -->

## License
Licensed under the [MIT License](LICENSE.md)  
Copyright (c) 2022 Chris Ludden