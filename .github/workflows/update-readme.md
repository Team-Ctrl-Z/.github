name: Update Profile README with Recent PRs

on:
schedule: - cron: '0 \* \* \* \*' # 매 시간마다 실행
workflow_dispatch:

jobs:
update-readme:
runs-on: ubuntu-latest

    steps:
      - name: Checkout .github
        uses: actions/checkout@v3

      - name: Install jq
        run: sudo apt-get install jq -y

      - name: Fetch recent PRs
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          echo "## 🔄 최근 PR (TEAM-CTRLZ)" > recent-prs.md
          N=5
          REPO="Team-Ctrl-Z/TEAM-CTRLZ"

          curl -s -H "Authorization: token $GH_TOKEN" "https://api.github.com/repos/$REPO/pulls?state=closed&sort=updated&direction=desc&per_page=$N" | \
          jq -r '.[] | select(.merged_at != null) |
            "- ✍️ [\(.title)](\(.html_url)) - by [@\(.user.login)](\(.user.html_url)) \(.merged_at)"' >> recent-prs.md

          # 시간 변환 (UTC → KST)
          cat recent-prs.md | while read line; do
            if [[ "$line" == *"https://github.com"* ]]; then
              utc=$(echo "$line" | grep -oP '\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}Z')
              kst=$(date -d "$utc 9 hours" "+%Y-%m-%d %H:%M (KST)")
              echo "$line" | sed "s/$utc/$kst/"
            else
              echo "$line"
            fi
          done > recent-prs-fixed.md

      - name: Merge with static README
        run: |
          cat profile/template.md recent-prs-fixed.md > profile/README.md

      - name: Commit and push
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "github-actions[bot]@users.noreply.github.com"
          git add profile/README.md
          git commit -m "Update profile README with latest PRs" || echo "No changes to commit"
          git push
