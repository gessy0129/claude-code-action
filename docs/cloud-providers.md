# Cloud Providers

You can authenticate with Claude using any of these five methods:

1. Direct Anthropic API (default)
2. Amazon Bedrock with OIDC authentication
3. Google Vertex AI with OIDC authentication
4. Microsoft Foundry with OIDC authentication
5. Claude Platform on AWS (Anthropic-operated API with AWS authentication)

For detailed setup instructions for AWS Bedrock and Google Vertex AI, see the [official documentation](https://code.claude.com/docs/en/github-actions#for-aws-bedrock:).

**Note**:

- Bedrock, Vertex, and Microsoft Foundry use OIDC authentication exclusively
- AWS Bedrock automatically uses cross-region inference profiles for certain models
- For cross-region inference profile models, you need to request and be granted access to the Claude models in all regions that the inference profile uses
- Claude Platform on AWS routes to the Anthropic-operated API (same models and features as the direct Claude API), authenticated with AWS credentials (SigV4) or a workspace API key, and billed through AWS Marketplace. Unlike Bedrock, AWS only provides the authentication and billing layer. See the [Claude Code on Claude Platform on AWS docs](https://code.claude.com/docs/en/claude-platform-on-aws)

## Model Configuration

Use provider-specific model names based on your chosen provider:

```yaml
# For direct Anthropic API (default)
- uses: anthropics/claude-code-action@v1
  with:
    anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
    # ... other inputs

# For Amazon Bedrock with OIDC
- uses: anthropics/claude-code-action@v1
  with:
    use_bedrock: "true"
    claude_args: |
      --model anthropic.claude-4-0-sonnet-20250805-v1:0
    # ... other inputs

# For Google Vertex AI with OIDC
- uses: anthropics/claude-code-action@v1
  with:
    use_vertex: "true"
    claude_args: |
      --model claude-4-0-sonnet@20250805
    # ... other inputs

# For Microsoft Foundry with OIDC
- uses: anthropics/claude-code-action@v1
  with:
    use_foundry: "true"
    claude_args: |
      --model claude-sonnet-4-5
    # ... other inputs

# For Claude Platform on AWS
- uses: anthropics/claude-code-action@v1
  with:
    use_aws_platform: "true"
    claude_args: |
      --model claude-sonnet-4-5
    # ... other inputs
  env:
    ANTHROPIC_AWS_WORKSPACE_ID: ${{ vars.ANTHROPIC_AWS_WORKSPACE_ID }}
    AWS_REGION: us-east-1
```

## OIDC Authentication for Cloud Providers

AWS Bedrock, GCP Vertex AI, and Microsoft Foundry all support OIDC authentication.

```yaml
# For AWS Bedrock with OIDC
- name: Configure AWS Credentials (OIDC)
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
    aws-region: us-west-2

- name: Generate GitHub App token
  id: app-token
  uses: actions/create-github-app-token@v2
  with:
    app-id: ${{ secrets.APP_ID }}
    private-key: ${{ secrets.APP_PRIVATE_KEY }}

- uses: anthropics/claude-code-action@v1
  with:
    use_bedrock: "true"
    claude_args: |
      --model anthropic.claude-4-0-sonnet-20250805-v1:0
    # ... other inputs

  permissions:
    id-token: write # Required for OIDC
```

```yaml
# For GCP Vertex AI with OIDC
- name: Authenticate to Google Cloud
  uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
    service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}

- name: Generate GitHub App token
  id: app-token
  uses: actions/create-github-app-token@v2
  with:
    app-id: ${{ secrets.APP_ID }}
    private-key: ${{ secrets.APP_PRIVATE_KEY }}

- uses: anthropics/claude-code-action@v1
  with:
    use_vertex: "true"
    claude_args: |
      --model claude-4-0-sonnet@20250805
    # ... other inputs

  permissions:
    id-token: write # Required for OIDC
```

```yaml
# For Microsoft Foundry with OIDC
- name: Authenticate to Azure
  uses: azure/login@v2
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

- name: Generate GitHub App token
  id: app-token
  uses: actions/create-github-app-token@v2
  with:
    app-id: ${{ secrets.APP_ID }}
    private-key: ${{ secrets.APP_PRIVATE_KEY }}

- uses: anthropics/claude-code-action@v1
  with:
    use_foundry: "true"
    claude_args: |
      --model claude-sonnet-4-5
    # ... other inputs
  env:
    ANTHROPIC_FOUNDRY_BASE_URL: https://my-resource.services.ai.azure.com

permissions:
  id-token: write # Required for OIDC
```

```yaml
# For Claude Platform on AWS with OIDC (SigV4)
- name: Configure AWS Credentials (OIDC)
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
    aws-region: us-east-1

- name: Generate GitHub App token
  id: app-token
  uses: actions/create-github-app-token@v2
  with:
    app-id: ${{ secrets.APP_ID }}
    private-key: ${{ secrets.APP_PRIVATE_KEY }}

- uses: anthropics/claude-code-action@v1
  with:
    use_aws_platform: "true"
    claude_args: |
      --model claude-sonnet-4-5
    # ... other inputs
  env:
    ANTHROPIC_AWS_WORKSPACE_ID: ${{ vars.ANTHROPIC_AWS_WORKSPACE_ID }}
    AWS_REGION: us-east-1

permissions:
  id-token: write # Required for OIDC
```

## Microsoft Foundry Setup

For detailed setup instructions for Microsoft Foundry, see the [official documentation](https://docs.anthropic.com/en/docs/claude-code/microsoft-foundry).

## Claude Platform on AWS Setup

Claude Platform on AWS routes requests to the Anthropic-operated API while authenticating and billing through AWS. Set `use_aws_platform: "true"` and provide:

- `ANTHROPIC_AWS_WORKSPACE_ID` (**required**): the workspace ID from your AWS-linked Anthropic organization, sent on every request as the `anthropic-workspace-id` header.
- `AWS_REGION` (**required**): used to compute the base URL `https://aws-external-anthropic.{region}.api.aws`. Provide `ANTHROPIC_AWS_BASE_URL` instead to override the URL (for example, when routing through a proxy or LLM gateway).
- Authentication, one of:
  - **SigV4** (recommended for CI): AWS credentials supplied by `aws-actions/configure-aws-credentials@v4` via OIDC. Requires `permissions: id-token: write`.
  - **Workspace API key**: set `ANTHROPIC_AWS_API_KEY`. It is sent as `x-api-key` and takes precedence over SigV4.

Optional:

- `ANTHROPIC_AWS_BASE_URL`: override the computed base URL (proxy / gateway).
- `CLAUDE_CODE_SKIP_ANTHROPIC_AWS_AUTH: "1"`: send unsigned requests when a gateway signs them on your behalf.

For the AWS Marketplace subscription, workspace, and IAM setup that comes before this, see the [Claude Platform on AWS documentation](https://platform.claude.com/docs/en/build-with-claude/claude-platform-on-aws). For Claude Code specifics, see [Claude Code on Claude Platform on AWS](https://code.claude.com/docs/en/claude-platform-on-aws).
