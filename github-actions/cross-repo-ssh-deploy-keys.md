# Github Actions Cross-Repository Push using SSH Deploy Keys

🗓️ **Date**: 2026-02-28

## What I Learned

When setting up automated cross-repository synchronization (such as mirroring a private repository to a public "showcase" version), using `GITHUB_TOKEN` or a standard Personal Access Token (PAT) often fails due to restricted `repo` scopes for the `github-actions[bot]` context, throwing a `403 Permission Denied` error during a non-fast-forward push.

Instead of generating and maintaining a high-privilege PAT with full repository access (which creates a single point of failure and violates the Principle of Least Privilege), the most robust and secure approach is to use **SSH Deploy Keys**. By generating an Ed25519 SSH keypair, adding the public key to the destination repository as a Deploy Key (with write access), and storing the private key as a GitHub Secret in the source repository, the Action can push seamlessly and securely with exact scope.

Additionally, always be careful with actions like `cpina/github-action-push-to-another-repository` when syncing directories. If you set `source-directory: "."`, the Action may unintentionally copy the `.git` directory and the full private history over to the public mirror, leading to massive security leaks or immediate history divergence rejections. Always sync to a sterile `dist/` directory first using `rsync` filtering.

## Example

Here is the safest and most decoupled architectural pattern for cross-repo syncing:

```yaml
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: "Isolate Public Artifacts"
        run: |
          mkdir -p dist-showcase
          # Safely copy only allowed files, excluding .git and ignores
          rsync -av --exclude-from=.showcaseignore --exclude="dist-showcase" --exclude=".git" . dist-showcase/

      - name: "Push via SSH Deploy Key"
        uses: cpina/github-action-push-to-another-repository@main
        env:
          SSH_DEPLOY_KEY: ${{ secrets.SHOWCASE_DEPLOY_KEY }}
        with:
          source-directory: "dist-showcase"
          destination-github-username: "YourOrg"
          destination-repository-name: "destination-repo"
          target-branch: "main"
```

## Why It Matters

- **Security Posture (Zero Trust)**: SSH Deploy keys are repository-specific. If the key leaks, an attacker only gains access to that single target repository, unlike a PAT which compromises the user's entire portfolio.
- **Workflow Reliability**: Bypasses the strict and rotating permissions of OAuth tokens tied to the CLI or the default Actions runner token.
- **Code Leak Prevention**: Explicit filtering into a separate directory avoids exposing proprietary source code hidden in the `.git` history to the public.

## References

- [GitHub Docs: Managing Deploy Keys](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys)
- [CPina Push to Another Repository Action](https://github.com/cpina/github-action-push-to-another-repository)
