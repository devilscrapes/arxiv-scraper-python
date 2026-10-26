<p align="center"><a href="https://apify.com/DevilScrapes/arxiv-papers-scraper"><img src="assets/banner.svg" alt="arXiv Scraper — Search &amp; Export Paper Metadata — Python example" width="100%"></a></p>

# arXiv Scraper in Python: preprint metadata from any query into a DataFrame

<p align="center">
<a href="https://apify.com/DevilScrapes/arxiv-papers-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/arxiv-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/arxiv-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

Use this repository to run the **[arXiv Papers Scraper](https://apify.com/DevilScrapes/arxiv-papers-scraper)** on Apify from Python. You write a query in arXiv's search syntax, for example `cat:cs.AI` for the artificial intelligence listing or `abs:"graph neural" AND cat:q-bio.QM` for a cross-field topic, and receive one record per paper with its identifier, title, abstract, ordered authors, primary and secondary categories, DOI, journal reference, the authors' comment and the first and latest version timestamps. The request loop and feed parsing happen in the cloud. What lands on your disk is `results.json` and `results.csv`, shaped for pandas, a Jupyter notebook or a scheduled job.

> **Actor:** [arXiv Scraper — Search & Export Paper Metadata](https://apify.com/DevilScrapes/arxiv-papers-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [arxiv-scraper-javascript](https://github.com/devilscrapes/arxiv-scraper-javascript)

## 🔥 What you get

- **arXiv's own query language, passed through**: field prefixes like `cat:`, `ti:` and `abs:`, the operators `AND`, `OR` and `ANDNOT`, and `submittedDate:[... TO ...]` ranges all reach the official export API untouched.
- **Paging you do not write**: set `maxResults` up to 5,000 and `pageSize` up to 2,000; the Actor requests consecutive pages and stops early when a page comes back short.
- **Flat JSON instead of Atom XML**: namespaces are resolved for you, titles have their line breaks collapsed and authors arrive as an ordered list of names.
- **Optional fields stay honest**: `doi`, `journal_ref` and `comment` are `null` when the paper's metadata does not carry them, never an empty guess.
- **Transient failures retried with backoff**: a temporary error or a maintenance page is retried a few times before the run gives up and reports the failure.
- **Identifies itself**: requests carry a Devil Scrapes user agent, in line with arXiv's request that automated clients say who they are.

## 🎯 Use cases

### Seed a systematic literature review

Combine a category and an abstract phrase, add a `submittedDate` range, and export the result set. Screeners get a CSV with titles and abstracts side by side, and the `url` column lets them open each candidate directly.

### Plot submission volume per subfield

Pull a few thousand recent papers from a parent archive like `cs` or `stat`, convert `published` to a datetime and count by month and `primary_category`. Shifts in where people publish show up within a quarter.

### Build an abstract corpus for embeddings

The `summary` field is plain text, ready to chunk and embed. Store `arxiv_id` next to each vector so a retrieval answer can link back to the abstract page and the PDF.

### Watch for revisions and journal acceptance

Re-run the same query weekly from cron. A changed `updated` timestamp means a new version, and a `journal_ref` that went from `None` to a value tells you the work was accepted somewhere.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/arxiv-scraper-python.git
cd arxiv-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [arXiv Scraper — Search & Export Paper Metadata](https://apify.com/DevilScrapes/arxiv-papers-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

Without the command-line extras the integration fits in five lines. `apify-client` starts the run, waits for it and then streams the stored rows back to you.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/arxiv-papers-scraper").call(run_input={'searchQuery': 'cat:cs.AI', 'sortBy': 'submittedDate', 'sortOrder': 'descending', 'maxResults': 10, 'proxyConfiguration': {'useApifyProxy': False}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

`main.py` hands `input.json` to the Actor as is. The bundled query asks for the ten newest submissions in `cs.AI`, newest first, which takes seconds and costs a few cents. Swap in your own query string; keeping `maxResults` small while you tune the syntax saves money.

```json
{
  "searchQuery": "cat:cs.AI",
  "sortBy": "submittedDate",
  "sortOrder": "descending",
  "maxResults": 10,
  "proxyConfiguration": {
    "useApifyProxy": false
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `searchQuery` | string | yes | `"cat:cs.AI"` | — |
| `sortBy` | string | no | `"submittedDate"` | `relevance`, `lastUpdatedDate`, `submittedDate` |
| `sortOrder` | string | no | `"descending"` | `ascending`, `descending` |
| `maxResults` | integer | no | `50` | — |
| `pageSize` | integer | no | `50` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": false}` | — |

## 📤 Output

Each record is a single paper. `arxiv_id` keeps the version suffix (`v1`, `v2` and so on), and `url` and `pdf_url` point at that exact version. The two dates are the strings arXiv returns, in UTC, so `pd.to_datetime(..., utc=True)` parses them directly. For papers with pre-2007 identifiers such as `hep-th/...`, `arxiv_id` holds only the part after the slash, so join on `url` when old papers are in the mix.

| Field | Type | Always present |
|---|---|---|
| `arxiv_id` | string | yes |
| `url` | string | yes |
| `pdf_url` | string | no |
| `title` | string | yes |
| `summary` | string | no |
| `authors` | array | no |
| `primary_category` | string | no |
| `categories` | array | no |
| `doi` | string or null | no |
| `journal_ref` | string or null | no |
| `comment` | string or null | no |
| `published` | string | yes |
| `updated` | string | no |
| `scraped_at` | string | no |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result emitted | $0.0015 |

Prices as configured on 2026-08-20. The [Actor page](https://apify.com/DevilScrapes/arxiv-papers-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

A monthly breakdown with pandas (`pip install pandas`). It removes version suffixes so each paper counts once, counts submissions per month and primary category, reports how many already carry a DOI and writes `monthly_counts.csv`.

```python
import json

import pandas as pd

papers = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))
papers["paper"] = papers["url"].str.replace(r"v\d+$", "", regex=True)
papers = papers.drop_duplicates("paper")
papers["month"] = pd.to_datetime(papers["published"], utc=True).dt.strftime("%Y-%m")
papers["revised"] = papers["updated"] != papers["published"]

monthly = papers.groupby(["month", "primary_category"]).agg(
    papers=("paper", "count"),
    with_doi=("doi", "count"),
    revised=("revised", "sum"),
    avg_authors=("authors", lambda a: a.map(len).mean()),
)
monthly.round(1).to_csv("monthly_counts.csv")
print(monthly.round(1).to_string())
```

## ❓ FAQ

### Do I need an arXiv account or key?

No. arXiv's export API is open, and the only secret this script uses is your Apify token. Billing goes to your Apify account, not to arXiv.

### Can it download the PDFs or search inside full text?

Neither. Queries match metadata such as title, abstract, authors and categories, and the rows give you `pdf_url`. Fetch the files yourself with `requests` if you need them, and keep arXiv's bulk-access guidance in mind.

### How do I restrict results to a date range?

Put arXiv's range expression into the query, for example `cat:stat.ML AND submittedDate:[202601010000 TO 202603312359]`. Sorting by `submittedDate` ascending then walks the window from its start.

### Why does a large run end with fewer rows than `maxResults`?

Usually the query matched fewer papers than you asked for, and the Actor stops once a page returns short. If arXiv kept failing even after the retries, the run is marked failed, and the log says so.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/arxiv-papers-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [arXiv Scraper — Search & Export Paper Metadata on Apify Store](https://apify.com/DevilScrapes/arxiv-papers-scraper)
- [JavaScript version of this example](https://github.com/devilscrapes/arxiv-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [Crossref Works Scraper — DOIs, Authors & Citations](https://apify.com/DevilScrapes/crossref-works-scraper) — DOI-registered journal versions of the same research.
- [OpenAlex Works Scraper — Papers, Citations & Authors](https://apify.com/DevilScrapes/openalex-works-scraper) — citation counts and institutions across all disciplines.
- [ORCID Researcher Profiles Scraper](https://apify.com/DevilScrapes/orcid-researcher-profiles-scraper) — public researcher records to enrich author lists.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/arxiv-papers-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with arXiv or Cornell University.
