import os
import re
import ssl
import urllib.request
import xml.etree.ElementTree as ET
from datetime import datetime, timezone, timedelta
from email.utils import parsedate_to_datetime
from feedgen.feed import FeedGenerator
from bs4 import BeautifulSoup
from urllib.parse import urlparse

# Settings
FEEDS_FILE = 'feeds.txt'
OUTPUT_FILE = 'feed.xml'
MAX_ITEMS = 100  # Recommended 80-120
TZ_MSK = timezone(timedelta(hours=3))

# Disable SSL verification for simplicity
ssl._create_default_https_context = ssl._create_unverified_context

def fetch_feed(url):
    """Download and parse RSS/Atom feed."""
    try:
        req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0'})
        with urllib.request.urlopen(req, timeout=30) as response:
            return response.read()
    except Exception as e:
        print(f'Error loading {url}: {e}')
        return None

def parse_feed(content):
    """Extract items from RSS/Atom feed."""
    items = []
    try:
        root = ET.fromstring(content)
        # Find all entries (item for RSS, entry for Atom)
        for entry in root.iter():
            tag = entry.tag.split('}')[-1]
            if tag in ('item', 'entry'):
                item = {}
                for child in entry:
                    child_tag = child.tag.split('}')[-1]
                    text = child.text.strip() if child.text else ''
                    if child_tag == 'title':
                        item['title'] = text
                    elif child_tag == 'link':
                        # For RSS link - text, for Atom - attribute
                        if 'href' in child.attrib:
                            item['link'] = child.attrib['href']
                        elif text:
                            item['link'] = text
                    elif child_tag == 'description':
                        # Clean HTML
                        item['description'] = BeautifulSoup(text, 'html.parser').get_text()[:500]
                    elif child_tag == 'summary':
                        item['description'] = BeautifulSoup(text, 'html.parser').get_text()[:500]
                    elif child_tag == 'pubDate' or child_tag == 'published' or child_tag == 'updated':
                        try:
                            dt = parsedate_to_datetime(text)
                            item['published'] = dt.astimezone(TZ_MSK)
                        except:
                            pass
                    elif child_tag == 'guid' or child_tag == 'id':
                        item['id'] = text
                # Skip if required fields are missing
                if 'title' in item and 'link' in item:
                    if 'published' not in item:
                        item['published'] = datetime.now(timezone.utc).astimezone(TZ_MSK)
                    items.append(item)
    except Exception as e:
        print(f'Error parsing: {e}')
    return items

def clean_text(text):
    """Clean text from HTML and extra characters."""
    if not text:
        return ''
    soup = BeautifulSoup(text, 'html.parser')
    return re.sub(r'\s+', ' ', soup.get_text()).strip()

def main():
    # Read source list
    sources = []
    if os.path.exists(FEEDS_FILE):
        with open(FEEDS_FILE, 'r') as f:
            sources = [line.strip() for line in f if line.strip() and not line.startswith('#')]
    
    print(f'Loading {len(sources)} sources...')
    
    # Collect all items
    all_items = []
    seen_ids = set()
    
    for source in sources:
        print(f'Processing: {source}')
        content = fetch_feed(source)
        if content:
            items = parse_feed(content)
            print(f'  Found {len(items)} items')
            for item in items:
                # Create unique ID
                item_id = item.get('id', item['link'])
                # Check for duplicates
                if item_id not in seen_ids:
                    seen_ids.add(item_id)
                    item['source'] = urlparse(source).netloc
                    all_items.append(item)
        else:
            print(f'  Failed to load')
    
    # Sort by publication date
    all_items.sort(key=lambda x: x.get('published', datetime.now(TZ_MSK)), reverse=True)
    
    # Trim to limit
    all_items = all_items[:MAX_ITEMS]
    
    print(f'Total unique items: {len(all_items)}')
    
    # Create final RSS
    fg = FeedGenerator()
    fg.id('https://89207549995alexr-ops.github.io/pocketbook-rss/feed.xml')
    fg.title('Business News')
    fg.description('News aggregator: RBC, Kommersant')
    fg.link(href='https://89207549995alexr-ops.github.io/pocketbook-rss/', rel='self')
    fg.language('ru')
    
    for item in all_items:
        fe = fg.add_entry()
        fe.id(item.get('id', item['link']))
        fe.title(clean_text(item['title'])[:200])
        fe.link(href=item['link'])
        fe.description(clean_text(item.get('description', '')))
        fe.pubDate(item['published'])
        fe.author(name=item['source'])
    
    # Save file
    fg.rss_file(OUTPUT_FILE)
    print(f'Done! Saved {len(all_items)} items to {OUTPUT_FILE}')

if __name__ == '__main__':
    main()
