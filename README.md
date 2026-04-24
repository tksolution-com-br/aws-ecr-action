# AWS ECR Action

This Action allows you to create Docker images and push into a ECR repository.

## Parameters

| Parameter                      | Type      | Default            | Description                                                                                                                                                    |
| ------------------------------ | --------- | ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `access_key_id`                | `string`  |                    | Your AWS access key id                                                                                                                                         |
| `secret_access_key`            | `string`  |                    | Your AWS secret access key                                                                                                                                     |
| `account_id`                   | `string`  |                    | Your AWS Account ID                                                                                                                                            |
| `repo`                         | `string`  |                    | Name of your ECR repository                                                                                                                                    |
| `region`                       | `string`  |                    | Your AWS region                                                                                                                                                |
| `create_repo`                  | `boolean` | `false`            | Set this to true to create the repository if it does not already exist                                                                                         |
| `set_repo_policy`              | `boolean` | `false`            | Set this to true to set a IAM policy on the repository                                                                                                         |
| `repo_policy_file`             | `string`  | `repo-policy.json` | Set this to repository policy statement json file. only used if the set_repo_policy is set to true                                                             |
| `image_scanning_configuration` | `boolean` | `false`            | Set this to True if you want AWS to scan your images for vulnerabilities                                                                                       |
| `tags`                         | `string`  | `latest`           | Comma-separated string of ECR image tags (ex latest,1.0.0,)                                                                                                    |
| `dockerfile`                   | `string`  | `Dockerfile`       | The path to the Dockerfile to be used (e.g., path/to/Dockerfile)                                                                                               |
| `extra_build_args`             | `string`  | `""`               | Extra flags to pass to docker build (see docs.docker.com/engine/reference/commandline/build)                                                                   |
| `cache_from`                   | `string`  | `""`               | Images to use as cache for the docker build (see `--cache-from` argument docs.docker.com/engine/reference/commandline/build)                                   |
| `path`                         | `string`  | `.`                | Path to Dockerfile, defaults to the working directory                                                                                                          |
| `prebuild_script`              | `string`  |                    | Relative path from top-level to script to run before Docker build                                                                                              |
| `registry_ids`                 | `string`  |                    | : A comma-delimited list of AWS account IDs that are associated with the ECR registries. If you do not specify a registry, the default ECR registry is assumed |
| `environment`                  | `string`  |                    | Environment to deploy (prd, hml, dev). Automatically uses the correct base Docker image from ECR to avoid Docker Hub rate limits                               |
| `base_image`                   | `string`  |                    | (Advanced) Override base Docker image. If not set, uses environment-based image or default docker:23.0.6                                                       |

## Usage

```yaml
jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: docker://ghcr.io/tksolution-com-br/aws-ecr-action:latest
        with:
          access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          account_id: ${{ secrets.AWS_ACCOUNT_ID }}
          repo: docker/repo
          region: ap-northeast-2
          tags: latest,${{ github.sha }}
          create_repo: true
          image_scanning_configuration: true
          set_repo_policy: true
          repo_policy_file: repo-policy.json
```

### Using Environment-Based Configuration (Recommended)

The simplest way to avoid Docker Hub rate limits is to use the `environment` parameter. The action will automatically select the correct base image from your private ECR:

**For Production Environment:**

```yaml
jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: tksolution-com-br/aws-ecr-action@master
        with:
          access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          account_id: ${{ secrets.AWS_ACCOUNT_ID }}
          repo: docker/repo
          region: ap-northeast-2
          tags: latest,${{ github.sha }}
          environment: prd
```

**For Staging/Homologation Environment:**

```yaml
jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: tksolution-com-br/aws-ecr-action@master
        with:
          access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          account_id: ${{ secrets.AWS_ACCOUNT_ID }}
          repo: docker/repo
          region: ap-northeast-2
          tags: latest,${{ github.sha }}
          environment: hml
```

**Dynamic Environment Selection:**

```yaml
jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Determine environment
        id: env
        run: |
          if [ "${{ github.ref }}" == "refs/heads/main" ]; then
            echo "name=prd" >> $GITHUB_OUTPUT
          else
            echo "name=hml" >> $GITHUB_OUTPUT
          fi
      - uses: tksolution-com-br/aws-ecr-action@master
        with:
          access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          account_id: ${{ secrets.AWS_ACCOUNT_ID }}
          repo: docker/repo
          region: ap-northeast-2
          tags: latest,${{ github.sha }}
          environment: ${{ steps.env.outputs.name }}
```

### Advanced: Custom Base Image

If you need full control over the base image, you can still specify it directly:

**For Production Environment:**

```yaml
jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: tksolution-com-br/aws-ecr-action@master
        with:
          access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          account_id: ${{ secrets.AWS_ACCOUNT_ID }}
          repo: docker/repo
          region: ap-northeast-2
          tags: latest,${{ github.sha }}
          base_image: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.ap-northeast-2.amazonaws.com/docker-base:prd
```

### Internal Configuration

The action automatically maps environments to ECR images:

- `prd`, `prod`, `production` → `{account_id}.dkr.ecr.{region}.amazonaws.com/docker-base:prd`
- `hml`, `homolog`, `staging` → `{account_id}.dkr.ecr.{region}.amazonaws.com/docker-base:hml`
- `dev`, `develop` → `{account_id}.dkr.ecr.{region}.amazonaws.com/docker-base:hml`
- No environment specified → `docker:23.0.6` (default)

## Version References

If you don't want to use the latest version, you can point to any reference in the repo directly:

```yaml
- uses: tksolution-com-br/aws-ecr-action@master
# or
- uses: tksolution-com-br/aws-ecr-action@v3
# or
- uses: tksolution-com-br/aws-ecr-action@0589ad88c51a1b08fd910361ca847ee2cb708a30
```

## License

The MIT License (MIT)
