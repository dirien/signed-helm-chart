# PoC To Create A Keyless Signed OCI Helm Chart

## Requirements

- CR_PAT for the GitHub container registry

See [this guide](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
for more information on how to create a personal access token.

## Task

Task is a task runner / build tool that aims to be simpler and easier to use than, for example, GNU Make.

Since it’s written in Go, Task is just a single binary and has no other dependencies, which means you don’t need to mess
with any complicated install setups just to use a build tool.

Once installed, you just need to describe your build tasks using a simple YAML schema in a file called Taskfile.yml:

Our Taskfile.yml file looks like this:

```yaml
version: '3'

tasks:
  purge:
    cmds:
      - echo "Purging..."
      - rm -rf dist

  default:
    deps:
      - purge
    cmds:
      - mkdir dist
      - echo $CR_PAT | helm registry login ghcr.io --username $OWNER --password-stdin
      - helm package signed-helm-chart --destination dist
      - helm push dist/signed-helm-chart-*.tgz oci://ghcr.io/dirien/ > .digest
      - cat .digest | awk -F "[, ]+" '/Digest/{print $NF}'
      - cosign sign ghcr.io/dirien/signed-helm-chart@$(cat .digest | awk -F "[, ]+" '/Digest/{print $NF}')
    env:
      HELM_EXPERIMENTAL_OCI: 1
      COSIGN_EXPERIMENTAL: "true"
```

And has a purge and default task inside.

## Helm

Helm 3 supports OCI for package distribution. Chart packages are able to be stored and shared across OCI-based
registries.

Enabling OCI Support Currently OCI support is considered experimental.

In order to use the commands described below, please set `HELM_EXPERIMENTAL_OCI` in the environment:

```
export HELM_EXPERIMENTAL_OCI=1
```

We're going to use `helm package` to create a the chart package and with `helm push` to push it to the OCI registry. In
this case , we will use `ghcr.io` as the registry.

## Cosign

To use `cosign` to sign keyless our OCI chart, we need to create set `COSIGN_EXPERIMENTAL` in the environment of
our `Taskfile.yml` task `default` and of course have the `cosign` binary installed.

## GitHub Action

To use keyless signing in GitHub Actions, we need do following steps:

### Step 1: Enable OCI Support

Add permissions to add `id-token` to the `permissions` section.

```yaml
permissions:
  id-token: write
```

### Step 2: Install Cosign

Just add this GitHub Action to your workflow:

```yaml
- name: Install cosign
  uses: sigstore/cosign-installer@v1.4.1
  with:
    cosign-release: 'v1.4.1'
```

### Step 3: Call the Taskfile

To build and sign our chart, we need to call the `Taskfile.yml` file in the GitHub Action. Just add this GitHub to your
workflow:

```yaml
- name: Build the helm chart and sign the oci image
  run: task
  env:
    CR_PAT: ${{ secrets.CR_PAT }}
    OWNER: ${{ github.repository_owner }}
```

## Result

If everything works, you should see the following output:

```shell
Run task
task: [purge] echo "Purging..."
Purging...
task: [purge] rm -rf dist
task: [default] mkdir dist
task: [default] echo $CR_PAT | helm registry login ghcr.io --username $OWNER --password-stdin
Login Succeeded
task: [default] helm package signed-helm-chart --destination dist
Successfully packaged chart and saved it to: dist/signed-helm-chart-0.1.0.tgz
task: [default] helm push dist/signed-helm-chart-*.tgz oci://ghcr.io/dirien/ > .digest
task: [default] cat .digest | awk -F "[, ]+" '/Digest/{print $NF}'
sha256:ab513674fdfdb100b9187747d63fca8cb9e89ba02977f3ef105ceee3ff9e9062
task: [default] cosign sign ghcr.io/dirien/signed-helm-chart@$(cat .digest | awk -F "[, ]+" '/Digest/{print $NF}')
Generating ephemeral keys...
Retrieving signed certificate...
client.go:196: root pinning is not supported in Spec 1.0.19
Successfully verified SCT...
tlog entry created with index: 955563
Pushing signature to: ghcr.io/dirien/signed-helm-chart
```

## Resources

- [go-task](https://taskfile.dev/)
- [cosign](https://github.com/sigstore/cosign)
- [helm](https://helm.sh/)
- [GitHub Actions](https://docs.github.com/en/actions)
- [Zero-friction “keyless signing” with Github Actions](https://chainguard.dev/posts/2021-12-01-zero-friction-keyless-signing)