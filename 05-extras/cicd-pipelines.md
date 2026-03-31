## Configure CI/CD Gates

**Recommended if you are using GitOps and want to prevent broken manifests or accidental secrets from being committed. Can be skipped if you are just experimenting.**

A CI pipeline on every pull request catches two categories of problems before they reach the cluster:

**Manifest validation** - builds your full manifest with `kustomize build`, renders any Helm charts inline, and schema-validates every resource against the Kubernetes API spec. If a resource reference is wrong, a value is missing, or a field doesn't exist in the spec, the PR fails before anything merges. Tools like [kubeconform](https://github.com/yannh/kubeconform) handle the schema validation step.

**Security scanning** - catches hardcoded secrets committed by accident (API keys, tokens, passwords), and flags common Kubernetes misconfigurations like privileged containers or missing security contexts. [gitleaks](https://github.com/gitleaks/gitleaks) and [Trivy](https://aquasecurity.github.io/trivy) are commonly used for this, but there are many alternatives.

Both GitHub Actions and GitLab CI support running these as jobs on pull requests. Configure them to block merges unless both checks pass. The specific tools are up to you; here is a minimal GitHub Actions example using kubeconform for manifest validation and gitleaks for secret scanning as a starting point:

```yaml
# .github/workflows/validate.yml
name: Validate

on:
  pull_request:

jobs:
  manifests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install kustomize
        run: |
          curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
          sudo mv kustomize /usr/local/bin/

      - name: Install kubeconform
        run: |
          curl -sL https://github.com/yannh/kubeconform/releases/latest/download/kubeconform-linux-amd64.tar.gz | tar xz
          sudo mv kubeconform /usr/local/bin/

      - name: Validate
        run: |
          kustomize build clusters/prod --load-restrictor LoadRestrictionsNone | \
          kubeconform -strict -kubernetes-version <KUBERNETES_VERSION>  # use the current stable version - find it at https://kubernetes.io/releases/ (e.g. "1.32.0")

  secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Add more jobs to this file as needed: image scanning, additional linters, policy checks. The pattern is always the same: run on `pull_request`, fail on non-zero exit code, require the job to pass before merge.

> **Reference:** [GitHub Actions](https://docs.github.com/en/actions) | [GitLab CI/CD](https://docs.gitlab.com/ee/ci/)

If your CI scanner pulls private container images, store your registry read token as a repository secret (**Repository → Settings → Secrets and variables → Actions**) and reference it as `${{ secrets.YOUR_SECRET_NAME }}` in the workflow.

**Optional:** Tools like [Dependabot](https://docs.github.com/en/code-security/dependabot) (GitHub) or [Renovate](https://docs.renovatebot.com/) automatically open PRs when your GitHub Actions versions, Helm chart versions, or other dependencies have updates, a low-effort way to stay current without manually tracking version numbers.