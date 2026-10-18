<p align="center"><a href="https://apify.com/DevilScrapes/rss-feed-scraper"><img src="assets/banner.svg" alt="RSS Feed Scraper — Atom, Podcast &amp; Multi-Feed Aggregator — Python example" width="100%"></a></p>

# RSS Feed Scraper in Python: RSS and Atom feeds into a pandas-ready table

<p align="center">
<a href="https://apify.com/DevilScrapes/rss-feed-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/rss-feed-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/rss-feed-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

This repository is a small Python client for the **[RSS Feed Scraper](https://apify.com/DevilScrapes/rss-feed-scraper)** on Apify. You give it feed URLs, whether that is a newspaper section, a company blog, a podcast or a GitHub releases page, and it returns every item as a row with the same columns no matter which feed dialect the publisher chose. Fetching and parsing happen on Apify, so there is no XML library to babysit and no special case for the one blog that formats dates its own way. What you get locally is `results.json` and `results.csv`, the shape a notebook, a DataFrame or a nightly cron archive wants.

> **Actor:** [RSS Feed Scraper — Atom, Podcast & Multi-Feed Aggregator](https://apify.com/DevilScrapes/rss-feed-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [rss-feed-scraper-javascript](https://github.com/devilscrapes/rss-feed-scraper-javascript)

## 🔥 What you get

- **Dialect-blind rows**: RSS 2.0, RSS 1.0 and Atom entries land in the same columns, and `feed_format` records which one each row came from.
- **The article body when the feed carries it**: `content_html` holds the full HTML from `content:encoded` or Atom `content`, and falls back to the summary for feeds that only ship a teaser. Turn it off with `includeContent` for lighter rows.
- **Dates you can sort**: `published` and `updated` are normalised to ISO-8601, and a date the parser cannot read becomes `null` instead of a stray string.
- **Many feeds, one run**: pass a list of URLs, cap each with `maxItemsPerFeed`, and items repeated under the same GUID or Atom id are kept once.
- **A broken URL stays a local problem**: temporary errors are retried with backoff, a feed that still fails is named in the run's status message, and every other feed is delivered.
- **Optional Apify Proxy and a custom User-Agent**, for the occasional publisher that refuses direct requests or wants to know who is reading.

## 🎯 Use cases

### Build a research corpus from a reading list

Put the feeds you follow for a topic into `input.json`, run the script and load the rows into pandas. With `content_html` switched on you have article bodies to clean, tokenise or summarise, each tagged with its source and a parsed publish date.

### Keep an archive that outlives the feed

A feed usually lists only its recent entries, not the full back catalogue. Schedule `python main.py --out archive/$(date +%F)` from cron, then concatenate the daily files and drop duplicates on `item_id` or `link`. After a few months you have history the publisher never exposed.

### Measure how often sources publish

Group rows by `feed_title` and resample on `published` to see posting cadence per outlet or per competitor blog. It is a quick way to spot a source that went quiet or one that doubled its output.

### Track topics through tags

`categories` is a list per row, so `explode` it and count. Over a few runs the counts show which subjects a publication is leaning into, without reading every headline.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/rss-feed-scraper-python.git
cd rss-feed-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [RSS Feed Scraper — Atom, Podcast & Multi-Feed Aggregator](https://apify.com/DevilScrapes/rss-feed-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

Without argument parsing and file writing, the integration is five lines. The official `apify-client` package starts the run, waits for it to complete and iterates over the items it produced.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/rss-feed-scraper").call(run_input={'feedUrls': ['https://news.ycombinator.com/rss', 'https://github.com/apify/apify-client-js/releases.atom'], 'maxItemsPerFeed': 5, 'includeContent': True, 'proxyConfiguration': {'useApifyProxy': False}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

The script hands `input.json` to the Actor without changes. The example reads two feeds of different dialects, an RSS front page and an Atom releases feed, and takes five items from each, so a first run returns at most ten rows. Add URLs to `feedUrls` and raise `maxItemsPerFeed` (up to 1,000) once the output looks right.

```json
{
  "feedUrls": [
    "https://news.ycombinator.com/rss",
    "https://github.com/apify/apify-client-js/releases.atom"
  ],
  "maxItemsPerFeed": 5,
  "includeContent": true,
  "proxyConfiguration": {
    "useApifyProxy": false
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `feedUrls` | array | yes | `["https://news.ycombinator.com/rss"]` | — |
| `maxItemsPerFeed` | integer | no | `50` | — |
| `includeContent` | boolean | no | `true` | — |
| `userAgent` | string | no | `"DevilScrapesBot/1.0 (+https://apify.com/DevilScrapes)"` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": false}` | — |

## 📤 Output

Every row is one feed item, carrying both its own fields and a few about the feed it came from. Columns that a publisher leaves out, such as an author or an update date, are `None`, and `categories` is always a list, possibly empty, so pandas does not have to guess types.

| Field | Type | Always present |
|---|---|---|
| `feed_url` | string | yes |
| `feed_title` | string or null | no |
| `feed_format` | string | yes |
| `item_id` | string or null | no |
| `title` | string | yes |
| `link` | string | yes |
| `author` | string or null | no |
| `summary` | string or null | no |
| `content_html` | string or null | no |
| `categories` | array | no |
| `published` | string or null | no |
| `updated` | string or null | no |
| `scraped_at` | string | no |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.005 |
| Result emitted | $0.001 |

Prices as configured on 2026-10-16. The [Actor page](https://apify.com/DevilScrapes/rss-feed-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

This pandas example (`pip install pandas`) parses the timestamps, prints a per-feed summary with item counts and date ranges, lists the most frequent tags and writes a newest-first `timeline.csv`.

```python
import json

import pandas as pd

items = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))
items["published"] = pd.to_datetime(items["published"], utc=True, errors="coerce")

per_feed = (
    items.groupby(["feed_url", "feed_format"])
    .agg(items=("link", "size"), newest=("published", "max"), oldest=("published", "min"))
    .reset_index()
)
print(per_feed.to_string(index=False))

tags = items["categories"].explode().dropna().value_counts().head(15)
print(tags.to_string())

timeline = items.drop_duplicates("link").sort_values("published", ascending=False)
timeline[["published", "feed_title", "title", "author", "link"]].to_csv("timeline.csv", index=False)
```

## ❓ FAQ

### Why not just use a feed parsing library myself?

You can, and for one well-behaved feed you should. The Actor earns its keep when the list grows: mixed dialects, retries, per-feed failures and a scheduled run that writes a dataset somewhere other than your laptop are already handled.

### Can I schedule it from a notebook or cron?

Yes. Call `main.main(["input.json", "--out", "data"])` from a notebook cell, or run `python main.py` from cron, Airflow or CI. A failed run exits with code 1, so the scheduler can alert on it.

### Does it return items from older pages of a feed?

No. It reads the document at each URL and does not follow `rel="next"` links. If a site offers paged feeds, add each page URL to `feedUrls` yourself.

### Will a podcast feed give me the audio file?

Episodes come through as rows with title, link, author and date, but there is no separate column for the enclosure URL. Use the rows as an episode index rather than a download list.

### How are duplicates handled across days?

Within one run, repeated GUIDs or Atom ids are kept once. Between runs nothing is remembered, which is why the archive use case above deduplicates on `item_id` after concatenating files.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/rss-feed-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [RSS Feed Scraper — Atom, Podcast & Multi-Feed Aggregator on Apify Store](https://apify.com/DevilScrapes/rss-feed-scraper)
- [JavaScript version of this example](https://github.com/devilscrapes/rss-feed-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [Google News Scraper — Search & Topic RSS Feeds](https://apify.com/DevilScrapes/google-news-scraper) — search and topic results from Google News when a site has no feed of its own.
- [Hacker News Scraper](https://apify.com/DevilScrapes/hacker-news-scraper) — Hacker News stories with score and comment counts that the RSS feed leaves out.
- [Substack Newsletter Scraper](https://apify.com/DevilScrapes/substack-newsletter-scraper) — a Substack archive with reactions, word counts and paywall flags.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/rss-feed-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with any publisher whose feed it reads.
