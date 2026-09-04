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

Python RSS Parser & Index Matcherscripts/ingest_and_match.pyimport os
import json
import base64
import urllib.request
import xml.etree.ElementTree as ET
from datetime import datetime

# Configuration
REPO = os.environ.get("GITHUB_REPOSITORY")  # e.g., "username/repo"
TOKEN = os.environ.get("GITHUB_TOKEN")
RSS_FEED_URL = "https://news.google.com/rss/search?q=disaster+remediation+emergency+tech&hl=en-US&gl=US&ceid=US:en" # Example feed

def fetch_github_index():
    url = f"https://api.github.com/repos/{REPO}/contents/index.md" # Adjust path to your master index
    headers = {"Accept": "application/vnd.github.v3+json"}
    if TOKEN:
        headers["Authorization"] = f"token {TOKEN}"
    
    try:
        req = urllib.request.Request(url, headers=headers)
        with urllib.request.urlopen(req) as response:
            data = json.loads(response.read().decode())
            return base64.b64decode(data['content']).decode('utf-8')
    except Exception as e:
        print(f"Warning: Could not fetch remote index ({e}), using local fallback.")
        return "Disaster remediation, systemic resilience, emergency tech."

def parse_rss_feed(feed_url):
    req = urllib.request.Request(feed_url, headers={"User-Agent": "Mozilla/5.0"})
    with urllib.request.urlopen(req) as response:
        xml_data = response.read()
    
    root = ET.fromstring(xml_data)
    items = []
    # Simple RSS parsing for items
    for item in root.findall('./channel/item'):
        title = item.find('title').text if item.find('title') is not None else ""
        link = item.find('link').text if item.find('link') is not None else ""
        pub_date = item.find('pubDate').text if item.find('pubDate') is not None else ""
        items.append({"title": title, "link": link, "date": pub_date})
    return items

def generate_outreach_draft(index_content, news_items):
    timestamp = datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S UTC")
    
    draft = f"""---\nrepository: momentum-mechanics/disaster-remediation-outreach\ntags: [automated-outreach, disaster-remediation, dynamic-sync]\ngenerated_at: {timestamp}\n---\n\n"""
    draft += f"# Automated Disaster Remediation Brief & Outreach Draft\n\n"
    draft += f"## Context Matrix\nIntegrated from repository master index and live global feeds.\n\n"
    draft += f"## Current Global Signals & Headlines\n"
    
    for item in news_items[:5]:  # Top 5 items
        draft += f"- **{item['title']}** ([Link]({item['link'])) - *{item['date']}*\n"
        
    draft += f"\n## Targeted Outreach Focus\n"
    draft += f"Based on current conditions and system architecture standards, drafting outreach targeting disaster remediation candidates.\n"
    
    return draft

if __name__ == "__main__":
    print("Fetching master index...")
    index_text = fetch_github_index()
    
    print("Parsing RSS feed...")
    feed_items = parse_rss_feed(RSS_FEED_URL)
    
    print("Generating outreach draft...")
    draft_content = generate_outreach_draft(index_text, feed_items)
    
    # Ensure drafts directory exists
    os.makedirs("drafts", exist_ok=True)
    filename = f"drafts/outreach-{datetime.utcnow().strftime('%Y%m%d-%H%M%S')}.md"
    
    with open(filename, "w", encoding="utf-8") as f:
        f.write(draft_content)
        
    print(f"Successfully generated draft: {filename}")
