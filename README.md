name: Automated Outreach Ingestion

on:
  schedule:
    - cron: '0 8 * * *' # Runs daily at 08:00 UTC
  workflow_dispatch: # Allows manual trigger from GitHub UI

jobs:
  build-drafts:
    runs-on: ubuntu-latest
    permissions:
      contents: write # Needed to commit generated files back to repo

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'

      - name: Run Ingestion and Matching Script
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITHUB_REPOSITORY: ${{ github.repository }}
        run: |
          python scripts/ingest_and_match.py

      - name: Commit and Push Generated Drafts
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "github-actions[bot]@users.noreply.github.com"
          git add drafts/
          git diff --quiet && git diff --staged --quiet || (git commit -m "chore: auto-generate fresh disaster remediation outreach draft" && git push)
# .github-workflows-automated-outreach.yml
GitHub Actions Workflow
