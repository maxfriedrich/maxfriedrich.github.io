---
layout: post
title: Building Python Lambda Functions in CDK with uv
---

(Jump directly to the code on GitHub: [https://github.com/maxfriedrich/uv-lambda-cdk-example](https://github.com/maxfriedrich/uv-lambda-cdk-example))

This post shows how to deploy AWS Lambda Python functions from a [uv](https://docs.astral.sh/uv/) [workspace](https://docs.astral.sh/uv/concepts/projects/workspaces/) with [CDK](https://docs.aws.amazon.com/cdk/v2/guide/home.html).
We build a custom construct based on `uv sync` that can be used as a replacement for the [aws-lambda-python-alpha `PythonFunction` construct](https://github.com/aws/aws-cdk/tree/main/packages/%40aws-cdk/aws-lambda-python-alpha).

Update 2025-02-15: with a small hack it's also possible to build Docker Lambda assets with this approach, see [Docker Lambda Functions](#docker-lambda-functions).

## Setup

In our example, we use a workspace with a layout like this:

```
.
├── cdk
│   ├── app.py
│   ├── cdk.json  # all default options
│   ├── pyproject.toml
│   └── python_lambda_function.py
├── packages
│   ├── demo-common  # common code used in the Lambda packages
│   │   ├── pyproject.toml
│   │   └── src
│   │       └── demo_common
│   │           └── __init__.py
│   ├── demo-lambda1  # package with a Lambda handler in lambda_function.py
│   │   ├── pyproject.toml
│   │   └── src
│   │       └── demo_lambda1
│   │           ├── __init__.py
│   │           └── lambda_function.py
│   └── demo-lambda2 
│       └── ...
├── pyproject.toml
└── uv.lock
```

The main `pyproject.toml` configures the workspace:

```toml
[project]
name = "uv-cdk-demo"
version = "0.1.0"

[tool.uv.workspace]
members = ["packages/*", "cdk"]
```

The Lambda packages' `pyproject.toml` configure their dependencies, e.g.:

```toml
[project]
name = "demo-lambda1"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "demo-common",
    "orjson>=3.10.12",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.uv.sources]
demo-common = { workspace = true }
```

Note that the Lambda packages are Python **libraries** created with `uv init --lib demo-lambda1`, not **apps**.

We can use a single lockfile and virtual environment for local development because there are (fortunately) no conflicts between the packages' dependencies.
The Lambda assets will however only include the dependencies they need.

The CDK code is written in Python as well.
It can be executed from the same virtual environment, which is very convenient.
We use a custom `PythonLambdaFunction` construct to build Lambda `.zip`s.

## Building a Lambda `.zip`

With this setup, building a Lambda function `.zip` means syncing the dependencies for one package into a temporary location and using `lib/python3.11/site-packages` as the Lambda code asset:

```bash
UV_PROJECT_ENVIRONMENT=/tmp/... \
  uv sync --package {package_name} \
  --frozen \
  --no-dev \
  --no-editable \
  --python {python_version}
```

This command needs to be run on the **same platform and architecture** as the Lambda function, e.g. for a x86_64 Lambda function, we can use a x86_64 GitHub Actions runner on Ubuntu but not a x86_64 MacBook running macOS.
As far as I can tell, there is no way to use `uv sync` to create an environment for a different platform or architecture.
See [#8935](https://github.com/astral-sh/uv/issues/8935), [#9350](https://github.com/astral-sh/uv/issues/9350) for additional discussion regarding building Lambda assets.

In CDK, we can provide a local bundling class that attempts bundling and fails fast if it's not possible, like when the architecture doesn't match.
If local bundling is not possible, CDK falls back to Docker bundling.
On recent Docker Desktop, running images with different architectures is not a problem, on GitHub Actions you would need to [set up buildx](https://github.com/docker/setup-buildx-action) or use a runner of the same architecture as the Lambda functions.

As of December 2024, there is a [bug in CDK](https://github.com/aws/aws-cdk/issues/30239) that makes the `platform=...` parameter in `BundlingOptions` useless and always picks the system platform.
We can only work around this by **using a SHA Docker image tag for the platform we want**, so the component is less flexible than I'd like, i.e. we can't interpolate the Python version into an image string like `f"ghcr.io/astral-sh/uv:0.5-python{python_version}-bookworm-slim"`.

## Making the Lambda `.zip` reproducible for faster deployments

If you have many small Lambda functions in your CDK app, it is desirable to not re-deploy all of them there is no code change, mainly because of deployment speed.
For example, a Lambda function that is hooked up to an API Gateway and has some provisioned capacity will take around 2 minutes to deploy and especially when there are dependencies between stacks, time adds up quickly.

Reproducibility here means: building the exact same asset (byte-for-byte) and letting CDK know that it is the same (via the asset hash) so the Lambda function has an empty diff and can be left as-is.

First, we need to **hash the CDK asset by the output**, not input.
In our uv workspace layout with a `cdk` directory, we are using `..` as the input path, so any change in the project directory changes the input hash.
Since building the asset is very fast with uv, it is fine for us to always build and then compare the output hash.

When using the `uv sync` command above with a `aws_cdk.FileSystem.mkdtemp()` or `tempfile.mkdtemp()` directory, assets are not reproducible:

- Scripts like `bin/fastapi` contain the temporary path used for building in the shebang line.
These scripts are not part of the `.zip` but their checksums are mentioned in dist-info `RECORD` files.
- uv puts the build timestamp into `uv_cache.json`.

We can avoid these problems by using a **stable temporary directory** for each package like `/tmp/uv-demo-{package_name}-build` and by setting **`UV_NO_INSTALLER_METADATA=1`** (available since [uv 0.5.7](https://github.com/astral-sh/uv/releases/tag/0.5.7)).

## Making the Lambda function cold start faster with compiled bytecode

While [AWS officially recommends not to compile bytecode / include `__pycache__` directories in Lambda `.zip`s](https://docs.aws.amazon.com/lambda/latest/dg/python-package.html#python-package-pycache), it is often done in practice to reduce cold start time (although I haven't measured it yet myself).
When we compile bytecode with the correct Python version and architecture, the AWS Lambda runtime uses it, which can be verified by setting `PYTHONVERBOSE=1` and inspecting the logs.

To compile bytecode eagerly with uv, we can pass `uv sync ... --compile-bytecode`.
This creates `__pycache__` directories with `.pyc` files for all packages.

All `.pyc` files contain the temporary directory for debugging (figuring out which file bytecode corresponds to).
Since we made it stable before and it is not used when executing the code, we can ignore this.

The bytecode also contains the timestamp when files were last modified, which we can bypass by setting [`SOURCE_DATE_EPOCH`](https://reproducible-builds.org/docs/source-date-epoch/).
We can't set it to 0 (= January 1, 1970) because [hatchling](https://github.com/pypa/hatch/tree/master/backend) (uv's default build backend as of December 2024) complains that `.zip` only supports timestamps after 1980, so we picked an arbitrary value `444444444` (= February 1, 1984) in this example.
The value you choose does not matter because if `SOURCE_DATE_EPOCH` is set, the bytecode is created with hash-based instead of timestamp-based invalidation, so the timestamp is not written to the bytecode header.

If you're curious about bytecode, I recommend playing around with the [`show_pyc` script from coverage.py](https://github.com/nedbat/coveragepy/blob/master/lab/show_pyc.py) -- e.g. using

```bash
uv run \
  https://github.com/nedbat/coveragepy/raw/refs/heads/master/lab/show_pyc.py \
  packages/demo-lambda1/src/demo_lambda1/__pycache__/__init__.cpython-311.pyc
```

## Putting it all together

In a [CDK stack](https://github.com/maxfriedrich/uv-lambda-cdk-example/blob/main/cdk/app.py), we configure a Python Lambda function like this:

```python
PythonLambdaFunction(
    self,
    "Lambda1",
    package_name="demo-lambda1",
    handler="demo_lambda1.lambda_function.lambda_handler",
)
```

The [construct](https://github.com/maxfriedrich/uv-lambda-cdk-example/blob/main/cdk/python_lambda_function.py) is (slightly simplified):

```python
def build_asset_command_and_env(
    package_name: str,
    output_path: str,
    architecture: _Architecture,  # named tuple that maps between Lambda, Docker, and Python "spellings" of architectures
    python_version: str,  # e.g. "3.11"
) -> tuple[list[str], dict[str, str]]:
    # Always use the same path per package to ensure that paths in the output are stable
    tmp_path = os.path.join("/tmp", f"uv-demo-{package_name}-build")

    commands = [
        # Ensure we are on the correct architecture
        '[ "$(uname -m)" = {architecture} ]'.format(architecture=architecture.platform_machine),
        # Create a virtual environment with the package's dependencies
        "uv sync --package {package_name} --frozen --no-dev --no-editable --compile-bytecode --python {python_version}".format(
            package_name=shlex.quote(package_name), python_version=python_version
        ),
        # Copy the virtual environment's site packages to the output path
        "cp -r {src} {dest}".format(
            src=os.path.join(tmp_path, "lib", f"python{python_version}", "site-packages", "."),
            dest=os.path.join(output_path.rstrip("/"), ""),
        ),
    ]
    command = ["bash", "-c", " && ".join(commands)]

    env = {
        "UV_PROJECT_ENVIRONMENT": tmp_path,  # stable temporary path for reproducible dist-info and bytecode
        "UV_NO_INSTALLER_METADATA": "1",  # don't write uv data that includes timestamps to dist-info directories
        "UV_LINK_MODE": "copy",  # files should be copied, not linked
        "SOURCE_DATE_EPOCH": "444444444",  # for hash-based bytecode, 0 is not allowed because zip requires timestamps after 1980
    }
    return command, env


class PythonLambdaFunction(aws_lambda.Function):
    def __init__(
        self,
        scope: Construct,
        construct_id: str,
        path: str,
        package_name: str,
        handler: str,
        **kwargs,
    ):
        command, env = build_asset_command_and_env(
            package_name,
            output_path="/asset-output",
            architecture=ARCHITECTURE,
            python_version=PYTHON_VERSION,
        )

        super().__init__(
            scope,
            construct_id,
            code=aws_lambda.Code.from_asset(
                path,
                asset_hash_type=AssetHashType.OUTPUT,  # decide hash based on output (default: based on input)
                bundling=BundlingOptions(
                    image=DockerImage.from_registry(BUNDLING_DOCKER_IMAGE),
                    environment=env,
                    command=command,
                    # The platform we provide here is ignored by CDK https://github.com/aws/aws-cdk/issues/30239
                    platform=ARCHITECTURE.docker_architecture,
                    local=UvLocalBundling(package_name),
                ),
            ),
            handler=handler,
            runtime=LAMBDA_RUNTIME,
            architecture=ARCHITECTURE.lambda_architecture,
            **kwargs,
        )
```

The local bundling checks the system platform and architecture before attempting to run `uv sync`:

```python
@jsii.implements(ILocalBundling)
class UvLocalBundling:
    def __init__(self, package_name: str) -> None:
        self.package_name = package_name
        super().__init__()

    def try_bundle(self, output_dir: str, *args, **kwargs) -> bool:
        if sys.platform != "linux" or platform.machine() != ARCHITECTURE.platform_machine:
            return False

        try:
            command, env = build_asset_command_and_env(
                self.package_name,
                output_path=output_dir,
                architecture=ARCHITECTURE,
                python_version=PYTHON_VERSION,
            )
            run_command(command, env=env)
        except RuntimeError as e:
            return False

        return True
```

See [https://github.com/maxfriedrich/uv-lambda-cdk-example](https://github.com/maxfriedrich/uv-lambda-cdk-example) for the full code!
The repository also includes Github Actions integration and a script to call Lambda functions for testing.

## What is missing

The `PythonLambdaFunction` construct only supports uv workspaces where packages are Python libraries (`uv init --lib`), not apps because the `.zip` is built only using the `site-packages` directory.
To support Python apps (`uv init --app`), we would need to copy the package contents into the asset output directory afterwards.

The construct is written in Python, so it can't be used e.g. from TypeScript but my hunch is that it should not be too hard to translate (maybe with some LLM help).

Let me know if it works for you and if you use a component like this in your CDK setup!

##  Docker Lambda Functions

CDK builds Docker Lambda functions a little differently, which makes this approach not work out of the box.
Instead of performing the bundling ("creating the asset") step at synthesis time, CDK only computes the input path hash and uses it to decide if the function needs to be built and re-deployed.
Since the input path we use above is `..`, it will change and therefore re-deploy all the time.

The key to making CDK not re-deploy is making the input directory exactly represent the Docker image contents.
For this, we can almost exactly reuse the .zip asset object we build above ([PR](https://github.com/maxfriedrich/uv-lambda-cdk-example/pull/7)).

To create the Docker asset, we first create a .zip asset with `lambda_code.bind(scope)`, then copy the Dockerfile into the asset's directory in `cdk.out` and use it as an input directory for a Docker asset:

```python
def python_docker_lambda_code(...):
    asset = lambda_code.bind(scope)
    asset_dir = os.path.join(
        "cdk.out",
        f"asset.{asset.s3_location.object_key.removesuffix('.zip')}"
    )
    shutil.copy(dockerfile, asset_dir)
    return aws_lambda.Code.from_asset_image(
        directory=asset_dir,
        build_args={"PYTHON_VERSION": python_version},
        ...
    )
```

This is of course a little bit hacky and not how you're intended to use CDK but it works for me. YMMV!
