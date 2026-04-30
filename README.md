## ⚡ کلون کردن فایل های از رلیز یک رپو در رپوی شما و دانلود کردن آنها در محدودیت فعلی

1. در ریپوی خودتون یه فایل در این آدرس ایجاد کنید: `.github/workflows/gitget.yml`.
2. کد زیر رو توش پیست کنید و دقت کنید عبارات متغیر رو به نسبت کار خودتون باید تغییر بدید:

این: `SOURCE_OWNER/SOURCE_REPO` ریپوی سورس که قراره ازش کلون و دانلود کنید (e.g., `torvalds/linux`).

این: `v1.2.3` رلیز تگ مورد نظر، دقت کنید دقیق وارد بشه.

این: `my-file.zip` نام دقیق فایل یا فایل هایی که قرار کلون بشن.

و این: `assets/` پوشه ای که قرار فایلها توش دانلود بشه (can be `.` for root).

```yaml
name: gitget

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

3. فایل رو کامیت و ثبت کنید.
4. 4. بعد برید قسمت اکشن ها و این اکشن جدید رو اجرا کنید و صبر کنید کارش رو بکنه.
5. برگردید توی رپوی خودتون و فایل هاتون رو از اونجا براحتی دانلود کنید :)
