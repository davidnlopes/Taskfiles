# Taskfiles

Reusable [Task](https://taskfile.dev) tasks that you include in other repositories. Each vertical, such as Terraform, is one YAML file with a documented public API. That API is a set of variables you set and tasks you call.

## Requirements

- Task 3.47 or later.
- A root `Taskfile.yml` with `version: '3'`. Task rejects an include whose `version` differs from the root.

## Add the repository to a project

To add this repository as a Git submodule at `.taskfiles`, run:

```bash
git submodule add https://github.com/davidnlopes/Taskfiles.git .taskfiles
```

Clone projects that use it with `git clone --recurse-submodules`. In CI, enable recursive submodule checkout.

To update to a newer version, run:

```bash
git submodule update --remote .taskfiles
```

## Use a vertical

Include the vertical in your `Taskfile.yml`, set the variables you need, and call the tasks:

```yaml
version: '3'

vars:
  TERRAFORM_DIR: '{{.TERRAFORM_DIR | default "terraform"}}'
  TERRAFORM_WORKSPACE: '{{.TERRAFORM_WORKSPACE | default "my-project-dev"}}'

env:
  AWS_PROFILE: my-profile

includes:
  common:
    taskfile: .taskfiles/common.yml
    flatten: true
  tf: .taskfiles/terraform.yml
```

Then run:

```bash
task                                       # lists tasks
task tf:plan TERRAFORM_VAR_FILES=configs/dev.tfvars
task tf:apply TERRAFORM_AUTO_APPROVE=true
```

You choose each namespace in `includes`. This README uses `tf` and `go`, and it uses `docker` for the Docker vertical.

### Conventions

These rules apply to every vertical:

- Public variables are uppercase and use the vertical's prefix, for example `TERRAFORM_DIR`. Lowercase variables are internal and can change without notice.
- Optional variables have a default. Required variables have none, because their values depend on your project. Tasks stop before they run anything if a required variable is missing, and before their own commands if it is empty.
- Boolean variables accept only `true` or `false`. Any other value stops the task with an error.
- Variables that hold many values are space-separated strings. To keep a space inside a value, quote it, for example `DOCKER_BUILD_ARGS='TITLE="two words" VERSION=1'`. YAML lists are not supported.
- Tasks that run commands also accept an `*_ARGS` or `*_FLAGS` variable. Use it to pass flags that the vertical does not expose.

### Set variable values

Write each value in your Taskfile as `'{{.NAME | default "value"}}'`, in root `vars` or in the `vars` of an include. The command line and the environment can then override it.

Values resolve in this order, highest first:

1. `vars` on a `task:` call
2. The command line
3. The environment
4. The value in your Taskfile
5. The default in this repository

A plain `NAME: value` behaves differently. In the `vars` of an include, it beats the command line and the environment. In root `vars`, it beats the environment. It also beats the command line for any include that uses the `taskfile:` key.

An empty value on the command line falls back to the value in your Taskfile when you use the `default` form.

### Use a vertical more than once

To run the same vertical against several modules, include it under several namespaces:

```yaml
includes:
  api:
    taskfile: .taskfiles/golang.yml
    vars:
      GO_DIR: services/api
  worker:
    taskfile: .taskfiles/golang.yml
    vars:
      GO_DIR: services/worker
```

### Set the default task

`task` with no name runs `help` when you include `common.yml` with `flatten: true`. To run a different task by default, exclude the `default` task and define your own:

```yaml
includes:
  common:
    taskfile: .taskfiles/common.yml
    flatten: true
    excludes: [default]
  tf: .taskfiles/terraform.yml

tasks:
  default:
    cmds:
      - task: tf:plan
```

### Extend a task

To run your own step before a shared task, or to give it a project name, define a task that calls it. Add a `desc` to list your task in `task help`:

```yaml
tasks:
  deploy:
    desc: Deploy the infrastructure
    deps: [check-credentials]
    cmds:
      - task: tf:apply

  check-credentials:
    cmds:
      - aws sts get-caller-identity
```

## Tools (`common.yml`)

`common.yml` lists tasks and installs Go tools into a project-local directory, so every contributor and CI run uses the same versions. It needs only the Go toolchain. The verticals do not depend on it. To get the `task help` and `task tools` names, include it with `flatten: true`.

Declare the tools in `TOOLS` as `NAME=MODULE@VERSION`, where `NAME` is the binary that `go install` produces. To run a tool from your own task, call `tool-run`. It installs the tool first when the tool is missing or its entry in `TOOLS` changed:

```yaml
vars:
  TOOLS: '{{.TOOLS | default "stringer=golang.org/x/tools/cmd/stringer@v0.30.0"}}'

includes:
  common:
    taskfile: .taskfiles/common.yml
    flatten: true

tasks:
  generate:
    desc: Generate String methods for the Color type
    cmds:
      - task: tool-run
        vars:
          TOOL: stringer
          TOOL_ARGS: -type=Color
```

If a command needs the path to the tool, such as in a pipeline, add `tool-install` to `deps` with the same `TOOL` value. Then run `{{.ROOT_DIR}}/bin/stringer`, or the same path under your `TOOLS_DIR`.

The task stops with an error if `TOOL` is not in `TOOLS`. Run `task tools` to install every declared tool in parallel, and `task tools-clean` to remove them.

| Variable | Default | Description |
| --- | --- | --- |
| `TOOLS` | (empty) | Space-separated `NAME=MODULE@VERSION` entries. |
| `TOOLS_DIR` | `<root>/bin` | Install directory. Relative paths resolve against the directory of the root Taskfile. Add it to `.gitignore`. |
| `GO_BIN` | `go` | Go executable that runs `go install`. |
| `TOOL` | Required by `tool-install` and `tool-run` | Name of the tool to install or run. |
| `TOOL_ARGS` | (empty) | Arguments that `tool-run` passes to the tool. |

Use a fixed version. Changing an entry reinstalls that tool, but `@latest` never changes, so Task does not check for a newer release. `task tools-clean` removes only the declared tools and their records, so other files in `TOOLS_DIR`, such as `go:build` output, stay.

## Terraform (`terraform.yml`)

`terraform.yml` runs Terraform against one root module. It requires Terraform 1.4 or later. To use OpenTofu, set `TERRAFORM_BIN=tofu`.

| Task | Description |
| --- | --- |
| `init` | Initialise the root module. |
| `workspace` | Select `TERRAFORM_WORKSPACE`, creating it if missing. |
| `fmt` | Rewrite Terraform files to canonical format. |
| `lint` | Check formatting and validate the configuration. |
| `plan` | Show the changes Terraform would make. |
| `apply` | Apply changes. |
| `destroy` | Destroy all resources. |
| `lock` | Record provider checksums for `TERRAFORM_LOCK_PLATFORMS`. |

### Terraform variables

| Variable | Default | Description |
| --- | --- | --- |
| `TERRAFORM_BIN` | `terraform` | Terraform executable. |
| `TERRAFORM_DIR` | `.` | Root module directory, passed to `-chdir`. |
| `TERRAFORM_WORKSPACE` | (empty) | Workspace to select, created if missing. If empty, the tasks do not touch workspaces. |
| `TERRAFORM_VAR_FILES` | (empty) | Space-separated `.tfvars` files for `plan`, `apply`, and `destroy`. If empty, no `-var-file` flag is passed. |
| `TERRAFORM_AUTO_APPROVE` | `false` | If `true`, adds `-auto-approve` to `apply` and `destroy`. |
| `TERRAFORM_INIT_ARGS` | (empty) | Extra flags for `init`, for example `-backend-config=backend.hcl`. |
| `TERRAFORM_PLAN_ARGS` | (empty) | Extra flags for `plan`, for example `-lock=false`. |
| `TERRAFORM_APPLY_ARGS` | (empty) | Extra flags for `apply`. |
| `TERRAFORM_DESTROY_ARGS` | (empty) | Extra flags for `destroy`. |
| `TERRAFORM_LOCK_PLATFORMS` | `linux_amd64 linux_arm64 darwin_amd64 darwin_arm64 windows_amd64` | Platforms that `lock` records in `.terraform.lock.hcl`. |

Terraform runs with `-chdir=$TERRAFORM_DIR`, so it resolves relative paths in `TERRAFORM_VAR_FILES` and `TERRAFORM_*_ARGS` against `TERRAFORM_DIR`.

The variables use the `TERRAFORM_` prefix, not `TF_`. Terraform reads `TF_*` environment variables itself. For example, `TF_WORKSPACE` overrides workspace selection.

## Go (`golang.yml`)

`golang.yml` formats, lints, tests, and builds one Go module. It uses only the Go toolchain.

| Task | Description |
| --- | --- |
| `fmt` | Rewrite Go files with `gofmt -s`. |
| `tidy` | Add missing and remove unused module requirements. |
| `lint` | Check `gofmt -s` formatting and run `go vet`. |
| `test` | Run tests, writing `GO_COVER_PROFILE` when set. |
| `cover-html` | Run tests and render the coverage profile as HTML. |
| `build` | Build `GO_BUILD_PKG` into `GO_BUILD_OUTPUT`. |

### Go variables

| Variable | Default | Description |
| --- | --- | --- |
| `GO_BIN` | `go` | Go executable. `gofmt` comes from the same toolchain's `GOROOT`. |
| `GO_DIR` | `.` | Module directory. Every command runs from here. |
| `GO_BUILD_PKG` | `.` | Package that `build` compiles, for example `./cmd/api`. |
| `GO_BUILD_OUTPUT` | `<root>/bin/` | Output path. If it ends with `/`, Go names the binary after the package. |
| `GO_BUILD_OS` | (empty) | `GOOS` for `build` only. If empty, Go builds for the host OS. |
| `GO_BUILD_ARCH` | (empty) | `GOARCH` for `build` only. If empty, Go builds for the host architecture. |
| `GO_BUILD_CGO_ENABLED` | (empty) | `CGO_ENABLED` for `build` only, `0` or `1`. If empty, Go uses the value from the environment. |
| `GO_BUILD_FLAGS` | (empty) | Extra flags for `go build`, for example `-trimpath -ldflags="-s -w"`. |
| `GO_TEST_PKGS` | `./...` | Packages that `test` runs. |
| `GO_TEST_FLAGS` | (empty) | Extra flags for `go test`, for example `-v -race -tags=integration`. |
| `GO_COVER_PROFILE` | `cover.out` | Coverage profile that `test` writes. If empty, `test` does not collect coverage. |
| `GO_COVER_HTML` | `cover.html` | HTML report that `cover-html` writes. |

Commands run from `GO_DIR`, so Go resolves relative paths in `GO_BUILD_OUTPUT`, `GO_COVER_PROFILE`, and `GO_COVER_HTML` against `GO_DIR`.

`GO_BUILD_OS`, `GO_BUILD_ARCH`, and `GO_BUILD_CGO_ENABLED` apply only to `build`. If you export `GOOS` instead, `go test` and `go vet` also compile for that OS, and `go test` cannot run a binary built for another OS.

To build a second binary, call `build` again with other values:

```yaml
tasks:
  build-seed:
    cmds:
      - task: go:build
        vars:
          GO_BUILD_PKG: ./cmd/seed
          GO_BUILD_OUTPUT: '{{.ROOT_DIR}}/bin/seed'
```

## Docker (`docker.yml`)

`docker.yml` builds an image and pushes it to a registry. It requires Docker 23.0 or later, where `docker build` uses BuildKit. Run `docker login` before you push.

| Task | Description |
| --- | --- |
| `build` | Build the image and tag it locally. |
| `push` | Push the tags that `build` created to `DOCKER_REGISTRY`. |
| `build-push` | Build the image and push it to `DOCKER_REGISTRY` in one step. |

To test an image and then push the same image, run `build` and then `push`, with the same `DOCKER_REGISTRY`, `DOCKER_IMAGE`, and `DOCKER_TAGS`:

```yaml
vars:
  DOCKER_REGISTRY: '{{.DOCKER_REGISTRY | default "ghcr.io/my-org"}}'
  DOCKER_IMAGE: '{{.DOCKER_IMAGE | default "my-app"}}'

includes:
  docker: .taskfiles/docker.yml
```

```bash
task docker:build DOCKER_TAGS="1.2.0 latest"
task docker:push DOCKER_TAGS="1.2.0 latest"
```

To publish a multi-platform image, run `build-push`, because a local image store might not hold one. The builder must support every platform you list. To select a builder, pass `--builder` in `DOCKER_BUILD_FLAGS`:

```bash
task docker:build-push DOCKER_TAGS=1.2.0 DOCKER_BUILD_PLATFORMS="linux/amd64 linux/arm64"
```

### Docker variables

| Variable | Default | Description |
| --- | --- | --- |
| `DOCKER_IMAGE` | Required | Image name without registry or tag, for example `my-app`. |
| `DOCKER_REGISTRY` | Required by `push` and `build-push` | Registry and optional namespace, for example `ghcr.io/my-org`. If you set it for `build`, it prefixes the local tags. |
| `DOCKER_BIN` | `docker` | Docker executable. |
| `DOCKER_TAGS` | `latest` | Space-separated image tags. |
| `DOCKER_BUILD_CONTEXT` | `.` | Build context. |
| `DOCKER_BUILD_FILE` | (empty) | Dockerfile path, relative to the task's working directory. If empty, Docker uses `Dockerfile` in the build context. |
| `DOCKER_BUILD_TARGET` | (empty) | Build stage to stop at. If empty, Docker builds the last stage. |
| `DOCKER_BUILD_ARGS` | (empty) | Space-separated `KEY=VALUE` entries, passed as `--build-arg`. |
| `DOCKER_BUILD_PLATFORMS` | (empty) | Space-separated platforms, for example `linux/amd64 linux/arm64`. If empty, Docker builds for the host platform. |
| `DOCKER_BUILD_FLAGS` | (empty) | Extra flags for `docker build`, for example `--pull --no-cache`. |
| `DOCKER_PUSH_FLAGS` | (empty) | Extra flags for `docker push`, for example `--quiet`. |

The variables avoid names that the Docker CLI reads from the environment. For example, `DOCKER_CONTEXT` selects a Docker context, so the build context variable is `DOCKER_BUILD_CONTEXT`.
