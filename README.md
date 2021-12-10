# AWS federation for GitHub Actions

Terraform module to configure GitHub Actions as an IAM OIDC identity provider
in AWS. This enables GitHub Actions to access resources within an AWS account
without requiring long-lived credentials to be stored as GitHub secrets.

## 🔨 Getting started

### Requirements

* [Terraform] 1.0+

### Installation and usage

Refer to the [complete example] to view all the available configuration options.
The following snippet shows the minimum required configuration to create a
working OIDC connection between GitHub Actions and AWS.

```terraform
provider "aws" {
  region = var.region
}

module "aws_federation_github" {
  source  = "unfunco/oidc-github/aws"
  version = "0.1.0"
  
  github_organisation = "your-org"
  github_repository   = "your-repo"
}
```

The following demonstrates how to use GitHub Actions once the Terraform module
has been applied to your AWS account. The action receives a JSON Web Token (JWT)
from the GitHub OIDC provider and then requests an access token from AWS.

```yaml
jobs:
  caller-identity:
    name: Check caller identity
    permissions:
      id-token: write
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v2
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-region: ${{ secrets.AWS_REGION }}
        role-session-name: CheckCallerIdentity
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github
    - run: aws sts get-caller-identity
```

### Inputs

#### Required

| Name                   | Type     | Description               |
| ---------------------- | -------- | ------------------------- |
| `github_organisation`  | `string` | GitHub organisation name. |
| `github_repository`    | `string` | GitHub repository name.   |

#### Optional

| Name                    | Default    | Description                                                    |
| ----------------------- | ---------- | -------------------------------------------------------------- |
| `enabled`               | `true`     | Flag to enable/disable creation of resources.                  |
| `force_detach_policies` | `false`    | Flag to force detachment of policies attached to the IAM role. |
| `iam_policy_name`       | `"github"` | Name of the IAM policy to be assumed by GitHub.                |
| `iam_policy_path`       | `"/"`      | Path to the IAM policy.                                        |
| `iam_role_name`         | `"github"` | Name of the IAM role.                                          |
| `iam_role_path`         | `"/"`      | Path to the IAM role.                                          |
| `managed_policy_arns`   | `[]`       | List of managed policy ARNs to apply to the IAM role.          |
| `max_session_duration`  | `3600`     | Maximum session duration in seconds.                           |
| `tags`                  | `{}`       | Map of tags to be applied to all resources.                    |

### Outputs

| Name                   | Type     | Description               |
| ---------------------- | -------- | ------------------------- |
| `github_organisation`  | `string` | GitHub organisation name. |
| `github_repository`    | `string` | GitHub repository name.   |

### Obtaining the thumbprint

```bash
JWKS=$(curl -s https://token.actions.githubusercontent.com/.well-known/openid-configuration | jq -r '.jwks_uri')
echo $JWKS | awk -F[/:] '{print $4}'
```

```bash
# TODO: Return the last certificate only.
openssl s_client \
  -servername token.actions.githubusercontent.com \
  -connect token.actions.githubusercontent.com:443 \
  -showcerts 2>/dev/null </dev/null \
  | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' \
  | openssl x509 -in /dev/stdin -fingerprint -noout
```

## References

* [Configuring OpenID Connect in Amazon Web Services]
* [Creating OpenID Connect (OIDC) identity providers]
* [Obtaining the thumbprint for an OpenID Connect Identity Provider]

## License

© 2021 [Daniel Morris](https://unfun.co)  
Made available under the terms of the [Apache License 2.0].

[Apache License 2.0]: LICENSE.md
[Complete example]: examples/complete
[Configuring OpenID Connect in Amazon Web Services]: https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services
[Creating OpenID Connect (OIDC) identity providers]: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html
[Make]: https://www.gnu.org/software/make/
[Obtaining the thumbprint for an OpenID Connect Identity Provider]: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc_verify-thumbprint.html
[Terraform]: https://www.terraform.io
