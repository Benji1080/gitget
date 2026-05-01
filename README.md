## ⚡ کلون کردن فایل ها از رلیز یک رپو در رپوی شما و دانلود کردن آنها در شرایط فعلی که رلیزها کار نمیکنه

1. در ریپوی خودتون یه فایل در این آدرس ایجاد کنید: `github/workflows/gitget.yml`.
2. فایل رو ویرایش کنید و کد زیر رو بذارید داخلش.
3. فایل رو کامیت و ثبت کنید.
4. بعد برید قسمت اکشن ها و این اکشن جدید رو اجرا کنید.
5. لینک مستقیم فایلی که قراره دانلود کنید رو وارد کنید، چه از گیتهاب یا هرجا در اینترنت!
6. صبر کنید اکشن کارش تمام بشه.

نکته: اگر حجم فایل بیشتر از 100 مگابایت باشه بدلیل محدودیت گیتهاب اون رو به پارت های زیپ شده تقسیم میکنه که بعد از دانلود باید اکسترکت کنید.

```yaml
name: Mirror File Download

on:
  workflow_dispatch:
    inputs:
      url:
        description: 'Direct download URL of the file'
        required: true
        type: string

permissions:
  contents: write

jobs:
  mirror:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          persist-credentials: true

      - name: Download and process file
        id: download
        env:
          INPUT_URL: ${{ github.event.inputs.url }}
        run: |
          #!/bin/bash
          set -euo pipefail

          LOG_FILE="download_log.txt"
          DOWNLOAD_DIR="downloads"
          TMP_DIR=$(mktemp -d)

          # -----------------------------------------------------------
          # 1. Clear log every run
          # -----------------------------------------------------------
          > "$LOG_FILE"

          log() {
            echo "$(date -u +'%Y-%m-%dT%H:%M:%SZ') $1" | tee -a "$LOG_FILE"
          }

          log "[INFO] Starting download"
          log "[INFO] Input URL: $INPUT_URL"

          # -----------------------------------------------------------
          # 2. Download using curl with -J (respect Content-Disposition)
          #    and -O (use remote filename).  This is exactly what
          #    download managers do.
          # -----------------------------------------------------------
          cd "$TMP_DIR"

          # curl writes the response body to the chosen filename,
          # and prints only the effective filename to stdout.
          FILENAME=$(curl -L -J -O -w '%{filename_effective}' --progress-bar "$INPUT_URL" 2>curl_stderr.log)
          CURL_EXIT=$?
          # Append curl stderr (progress & errors) to main log
          cat curl_stderr.log >> "../$LOG_FILE"

          if [ $CURL_EXIT -ne 0 ]; then
            log "[ERROR] curl failed (exit code: $CURL_EXIT)"
            exit 1
          fi

          log "[INFO] Original filename determined by server: $FILENAME"
          cd - >/dev/null

          # Remove any previous file or split parts with the same name
          rm -f "$DOWNLOAD_DIR/$FILENAME" "$DOWNLOAD_DIR/$FILENAME.split.zip" "$DOWNLOAD_DIR/$FILENAME.split.z"*

          # Move the downloaded file into the downloads/ folder
          mkdir -p "$DOWNLOAD_DIR"
          mv "$TMP_DIR/$FILENAME" "$DOWNLOAD_DIR/$FILENAME"

          FILEPATH="$DOWNLOAD_DIR/$FILENAME"
          FILESIZE=$(stat -c %s "$FILEPATH")
          HUMAN_SIZE=$(numfmt --to=iec --suffix=B "$FILESIZE" 2>/dev/null || echo "$FILESIZE bytes")
          log "[INFO] Downloaded: $FILESIZE bytes ($HUMAN_SIZE)"

          # -----------------------------------------------------------
          # 3. Split if > 100 MB (standard multi‑part ZIP, 100 MB parts)
          # -----------------------------------------------------------
          MAX_SIZE=$((100 * 1024 * 1024))
          if [ "$FILESIZE" -gt "$MAX_SIZE" ]; then
            log "[INFO] File exceeds 100 MB, splitting into a multi‑part ZIP archive"
            cd "$DOWNLOAD_DIR"
            zip -0 -s 100m -r "${FILENAME}.split.zip" "$FILENAME" 2>&1 | tee -a "../$LOG_FILE"
            if [ ${PIPESTATUS[0]} -ne 0 ]; then
              log "[ERROR] ZIP split failed"
              exit 1
            fi
            rm -f "$FILENAME"   # delete original large file
            cd ..

            # Gather part names and sizes for the summary
            PARTS_INFO=""
            for part in "$DOWNLOAD_DIR/${FILENAME}.split.zip" "$DOWNLOAD_DIR/${FILENAME}.split.z"??; do
              [ -f "$part" ] || continue
              P_SIZE=$(stat -c %s "$part")
              P_HUMAN=$(numfmt --to=iec --suffix=B "$P_SIZE" 2>/dev/null || echo "$P_SIZE bytes")
              PARTS_INFO+="$(basename "$part")|${P_SIZE}|${P_HUMAN}\n"
            done

            echo "split=true" >> "$GITHUB_OUTPUT"
            echo "filename=$FILENAME" >> "$GITHUB_OUTPUT"
            echo "archive_basename=${FILENAME}.split" >> "$GITHUB_OUTPUT"
            echo "parts_list<<EOF" >> "$GITHUB_OUTPUT"
            echo -e "$PARTS_INFO" >> "$GITHUB_OUTPUT"
            echo "EOF" >> "$GITHUB_OUTPUT"
          else
            log "[INFO] File within 100 MB limit, keeping original"
            echo "split=false" >> "$GITHUB_OUTPUT"
            echo "filename=$FILENAME" >> "$GITHUB_OUTPUT"
            echo "filepath=$FILEPATH" >> "$GITHUB_OUTPUT"
          fi

          echo "filesize=$FILESIZE" >> "$GITHUB_OUTPUT"
          echo "human_size=$HUMAN_SIZE" >> "$GITHUB_OUTPUT"
          log "[INFO] Done"

      - name: Commit log and files to the repository
        if: always()
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add download_log.txt
          if [ -d downloads ]; then
            git add downloads/
          fi
          git diff --cached --quiet || git commit -m "Mirror download: ${{ github.event.inputs.url }}"
          git push

      - name: Create job summary
        if: always()
        env:
          REPO: ${{ github.repository }}
          BRANCH: ${{ github.ref_name }}
          SPLIT: ${{ steps.download.outputs.split }}
          FILENAME: ${{ steps.download.outputs.filename }}
          FILEPATH: ${{ steps.download.outputs.filepath }}
          ARCHIVE_BASENAME: ${{ steps.download.outputs.archive_basename }}
          FILESIZE: ${{ steps.download.outputs.filesize }}
          HUMAN_SIZE: ${{ steps.download.outputs.human_size }}
          PARTS_LIST: ${{ steps.download.outputs.parts_list }}
        run: |
          LOG_TEXT=$(cat download_log.txt 2>/dev/null || echo "No log file found")
          {
            echo "## Download Summary"
            echo ""
            if [ "$SPLIT" = "true" ]; then
              echo "**Split archive parts** (download all parts, then open the \`.zip\` file):"
              echo ""
              while IFS='|' read -r partname bytes human; do
                [ -z "$partname" ] && continue
                echo "- [\`$partname\` ($human)](https://raw.githubusercontent.com/${REPO}/${BRANCH}/downloads/$partname)"
              done <<< "$(echo -e "$PARTS_LIST")"
              echo ""
              echo "**Total original size:** \`$HUMAN_SIZE\`"
              echo ""
              echo "> **Reassembly:** Open \`${FILENAME}.split.zip\` with 7‑Zip, WinRAR, or WinZip. The \`.z01\`, \`.z02\` parts are read automatically."
            elif [ -n "$FILEPATH" ]; then
              echo "**Downloaded file:** [\`$FILENAME\` (${HUMAN_SIZE})](https://raw.githubusercontent.com/${REPO}/${BRANCH}/$FILEPATH)"
            else
              echo "❌ Download did **not** produce a usable file. Check the log below."
            fi
            echo ""
            echo "### Detailed Log"
            echo '```'
            echo "$LOG_TEXT"
            echo '```'
          } >> "$GITHUB_STEP_SUMMARY"
```
