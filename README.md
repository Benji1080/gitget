## ⚡ Mirror external github releases into your repo (and download):

1. In your repository, create a new file at `.github/workflows/copy-release-asset.yml`.
2. Paste the following YAML and replace the placeholders:
   - `SOURCE_OWNER/SOURCE_REPO` with the source repository (e.g., `torvalds/linux`).
   - `v1.2.3` with the release tag that holds the asset.
   - `my-file.zip` with the exact filename of the asset.
   - `assets/` with the folder where you want the file stored (can be `.` for root).

```yaml
name: Copy Release Asset

on:
  workflow_dispatch:   # allows you to trigger the copy manually from the Actions tab
  # schedule:          # optional: uncomment to run on a schedule (cron)
  #   - cron: '0 0 * * 0'

# Ensure the workflow has write access to the repo
permissions:
  contents: write

jobs:
  copy:
    runs-on: ubuntu-latest
    steps:
      # 1. Checkout your own repository so we can add the file
      - name: Checkout my repo
        uses: actions/checkout@v4

      # 2. Download the release asset from the other repository
      - name: Download release asset
        uses: robinraju/release-downloader@v1.11
        with:
          repository: "SOURCE_OWNER/SOURCE_REPO"   # the repo that has the release
          tag: "v1.2.3"                            # the release tag
          fileName: "my-file.zip"                  # the asset file name, multiple files with comma separated, all files with *
          out-file-path: "assets"                  # folder in your repo to store the file
          # token: ${{ secrets.SOURCE_TOKEN }}     # ONLY needed if the source repo is private

      # 3. Commit and push the new file to your repository
      - name: Commit and push
        uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: 'Add release asset from external repo'
          branch: main   # or the branch you want to commit to
```

3. Commit the workflow file to your default branch (usually `main`).

## 🚀 How to run it

1. Go to your repository on GitHub.
2. Click the **Actions** tab.
3. Select the **Copy Release Asset** workflow on the left.
4. Click the **Run workflow** button and then **Run workflow**.

The workflow will immediately download the asset from the source release and commit it to your repository. You’ll see the new file in your repo’s file tree once the run finishes.
