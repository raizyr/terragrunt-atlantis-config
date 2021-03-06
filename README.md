<p align="center">
  <img alt="Terragrunt Atlantis Config by Transcend" src="https://user-images.githubusercontent.com/7354176/78756035-f9863480-792e-11ea-96d3-d4ffe50e0269.png"/>
</p>
<h1 align="center">Terragrunt Atlantis Config</h1>
<p align="center">
  <strong>Generate Atlantis Config for Terragrunt projects.</strong>
</p>
<br />

## What is this?

[Atlantis](https://runatlantis.io) is an awesome tool for Terraform pull request automation. Each repo can have a YAML config file that defines Terraform module dependencies, so that PRs that affect dependent modules will automatically generate `terraform plan`s for those modules.

[Terragrunt](https://terragrunt.gruntwork.io) is a Terraform wrapper, which has the concept of dependencies built into its configuration.

This tool creates Atlantis YAML configurations for Terragrunt projects by:

- Finding all `terragrunt.hcl` in a repo
- Evaluating their "dependency" and "terraform" source blocks to find their dependencies
- Creating a Directed Acyclic Graph of all dependencies
- Constructing and logging YAML in Atlantis' config spec that reflects the graph

This is especially useful for organizations that use monorepos for their Terragrunt config (as we do at Transcend), and have thousands of lines of config.

## Installation and Usage

Recommended: Install any version via go get:

```bash
cd && GO111MODULE=on go get github.com/transcend-io/terragrunt-atlantis-config@master && cd -
```

Alternative: Install a stable versions via Homebrew:

```bash
brew install transcend-io/tap/terragrunt-atlantis-config
```

This module officially supports golang versions v1.13, v1.14, and v1.15

Usage Examples:

```bash
# From the root of your repo
terragrunt-atlantis-config generate

# or from anywhere
terragrunt-atlantis-config generate --root /some/path/to/your/repo/root

# output to a file
terragrunt-atlantis-config generate --autoplan --output ./atlantis.yaml

# enable auto plan
terragrunt-atlantis-config generate --autoplan

# enable auto merge
terragrunt-atlantis-config generate --automerge

# define the workflow
terragrunt-atlantis-config generate --workflow web --output ./atlantis.yaml

# ignore parent terragrunt configs (those which don't reference a terraform module)
terragrunt-atlantis-config generate --ignore-parent-terragrunt

# Enable the project name creation
terragrunt-atlantis-config generate --create-project-name
```

Finally, check the log output (or your output file) for the YAML.

## Extra dependencies

For 99% of cases, this tool can sniff out all dependencies in a module. However, you may have times when you want to add in additional dependencies such as:

- You use Terragrunt's `read_terragrunt_config` function in your locals, and want to depend on the read file
- Your Terragrunt module should be run anytime some non-terragrunt file is updated, such as a Dockerfile or Packer template
- You want to run _all_ modules any time your product has a major version bump
- You believe a module should be reapplied any time some other file or directory is updated

In these cases, you can customize the `locals` block in that Terragrunt module to have a field named `extra_atlantis_dependencies` with a list
of values you want included in the config, such as:

```hcl
locals {
  extra_atlantis_dependencies = [
    "some_extra_dep",
    find_in_parent_folders(".gitignore")
  ]
}
```

In your `atlantis.yaml` file, you will end up seeing output like:

```yaml
- autoplan:
    enabled: false
    when_modified:
    - '*.hcl'
    - '*.tf*'
    - some_extra_dep
    - ../../.gitignore
  dir: example-setup/extra_dependency
```

If you specify `extra_atlantis_dependencies` in the parent Terragrunt module, they will be merged with the child dependencies using the following rules:

1. Any function in a parent will be evaluated from the child's directory. So you can use `get_parent_terragrunt_dir()` and other functions like you normally would in terragrunt.
2. Absolute paths will work as they would in a child module, and the path in the output will be relative from the child module to the absolute path
3. Relative paths, like the string `"foo.json"`, will be evaluated as relative to the Child module. This means that if you need something relative to the parent module, you should use something like `"${get_parent_terragrunt_dir()}/foo.json"`

## All Flags

One way to customize the behavior of this module is through CLI flag values passed in at runtime. These settings will apply to all modules.

| Flag Name                    | Description                                                                                                                                | Default Value     |
|------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|-------------------|
| `--autoplan`                 | The default value for autoplan settings. Can be overriden by locals.                                                                       | false             |
| `--automerge`                | Enables the automerge setting for a repo.                                                                                                  | false             |
| `--ignore-parent-terragrunt` | Ignore parent Terragrunt configs (those which don't reference a terraform module).<br>In most cases, this should be set to `true`          | false             |
| `--parallel`                 | Enables `plan`s and `apply`s to happen in parallel. Will typically be used with `--create-workspace`                                       | true              |
| `--create-workspace`         | Use different auto-generated workspace for each project. Default is use default workspace for everything                                   | false             |
| `--create-project-name`      | Add different auto-generated name for each project                                                                                         | false             |
| `--preserve-workflows`       | Preserves workflows from old output files. Useful if you want to define your workflow definitions on the client side                       | true              |
| `--workflow`                 | Name of the workflow to be customized in the atlantis server. If empty, will be left out of output                                         | ""                |
| `--output`                   | Path of the file where configuration will be generated. Typically, you want a file named "atlantis.yaml". Default is to write to `stdout`. | ""                |
| `--root`                     | Path to the root directory of the git repo you want to build config for.                                                                   | current directory |
| `--terraform-version`        | Default terraform version to specify for all modules. Can be overriden by locals                                                           | ""                |

## All Locals

Another way to customize the output is to use `locals` values in your terragrunt modules. These can be set in either the parent or child terragrunt modules, and the settings will only affect the current module (or all child modules for parent locals).

| Locals Name                   | Description                                                                                                                                                    | type         |
|-------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------|
| `atlantis_workflow`           | The custom atlantis workflow name to use for a module                                                                                                          | string       |
| `atlantis_terraform_version`  | Allows overriding the `--terraform-version` flag for a single module                                                                                           | string       |
| `atlantis_autoplan`           | Allows overriding the `--autoplan` flag for a single module                                                                                                    | bool         |
| `atlantis_skip`               | If true on a child module, that module will not appear in the output.<br>If true on a parent module, none of that parent's children will appear in the output. | bool         |
| `extra_atlantis_dependencies` | See [Extra dependencies](https://github.com/transcend-io/terragrunt-atlantis-config#extra-dependencies)                                                        | list(string) |

## Auto Enforcement with Github Actions

It's a best practice to require that `atlantis.yaml` stays up to date on each Pull Request.

To make this easy, there is an open-source Github Action that will fail a status check on your PR if the `atlantis.yaml` file is out of date.

To use it, add this yaml to a new github action file in your repo:

```yaml
name: terragrunt-atlantis-config
on:
  push:
    paths:
    - '**.hcl'
    - '**.tf'
    - '**.hcl.json'

jobs:
  terragrunt_atlantis_config:
    runs-on: ubuntu-latest
    name: Validate atlantis.yaml
    steps:
      - uses: actions/checkout@v2
      - name: Ensure atlantis.yaml is up to date using terragrunt-atlantis-config
        id: atlantis_validator
        uses: transcend-io/terragrunt-atlantis-config-github-action@v0.0.3
        with:
          version: v0.11.1
          extra_args: '--autoplan --parallel=false
```

You can customize the version and flags you typically pass to the `generate` command in those final two lines.

## Separate workspace for parallel plan and apply

Atlantis added support for running plan and apply parallel in [v0.13.0](https://github.com/runatlantis/atlantis/releases/tag/v0.13.0).

To use this feature, projects have to be separated in different workspaces, and the `create-workspace` flag enables this by concatenating the project path as the
name of the workspace.

As an example, project `${git_root}/stage/app/terragrunt.hcl` will have the name `stage_app` as workspace name. This flag should be used along with `parallel` to enable parallel plan and apply:

```bash
terragrunt-atlantis-config generate --output atlantis.yaml --parallel --create-workspace
```

Enabling this feature may consume more resources like cpu, memory, network, and disk, as each workspace will now be cloned separately by atlantis.

As when defining the workspace this info is also needed when running `atlantis plan/apply -d ${git_root}/stage/app -w stage_app` to run the command on specific directory,
you can also use the `atlantis plan/apply -p stage_app` in case you have enabled the `create-project-name` cli argument (it is `false` by default).

## Contributing

To test any changes you've made, run `make test`.

Once all your changes are passing and your PR is reviewed, a merge into `master` will trigger a CircleCI job to build the new binary, test it, and deploy it's artifacts to an S3 bucket.

You can then open a PR on our homebrew tap similar to https://github.com/transcend-io/homebrew-tap/pull/4, and as soon as that merges your code will be released.

## License

[![FOSSA Status](https://app.fossa.io/api/projects/git%2Bgithub.com%2Ftranscend-io%2Fterragrunt-atlantis-config.svg?type=large)](https://app.fossa.io/projects/git%2Bgithub.com%2Ftranscend-io%2Fterragrunt-atlantis-config?ref=badge_large)
