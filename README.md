# One Pipe

**One pipeline that works for any data source.**

One Pipe is an [n8n](https://n8n.io) workflow that runs every 3 days. It collects data from several free public sources, rewrites every record into one shared schema, runs quality checks, and saves a clean CSV plus a JSON quality report.

The included version is set up for **marketing data**: ad copy, campaign performance and industry news. Only the collect step is marketing-specific. Swap the sources and the same pipe tracks competitors, research papers, hiring signals or customer reviews, with no changes to the cleaning, checks or report.

```
 ① Collect          ② Normalize           ③ Quality checks         ④ Save
┌──────────────┐   ┌────────────────┐   ┌────────────────────┐   ┌──────────────────────┐
│ 2 HF datasets│──▶│ map each source│──▶│ trim · dates →     │──▶│ one_pipe_records.csv │
│ 4 RSS feeds  │   │ to 16 columns  │   │ YYYY-MM-DD · drop  │   │ one_pipe_quality_    │
│ retry 3×,    │   │ log "Source    │   │ incomplete · dedupe│   │   report.json        │
│ never fail   │   │ unavailable"   │   │ · flag gaps        │   │                      │
└──────────────┘   └────────────────┘   └────────────────────┘   └──────────────────────┘
```

## Results from a real run

| | |
|---|---|
| Sources | 6 of 6 available |
| Records collected | 295 |
| Duplicates removed | 10 |
| Clean records | 285 |
| Fully complete | 269 (**94.4%**) |
| Runtime | ~3 seconds |

The full report from that run is in [`docs/sample-quality-report.json`](docs/sample-quality-report.json).

## Sources

| Source | What it provides | Type |
|---|---|---|
| [jaykin01/advertisement-copy](https://huggingface.co/datasets/jaykin01/advertisement-copy) | Product ads, split into headline, body and call to action | Hugging Face dataset |
| [federicomoreno/marketing-campaign-performance-200k](https://huggingface.co/datasets/federicomoreno/marketing-campaign-performance-200k) | Campaign type, channel, audience, conversion rate, ROI | Hugging Face dataset |
| [MarTech](https://martech.org/feed/) | Industry news | RSS |
| [Marketing Dive](https://www.marketingdive.com/feeds/news/) | Industry news | RSS |
| [HubSpot Marketing Blog](https://blog.hubspot.com/marketing/rss.xml) | Marketing how-tos | RSS |
| [Social Media Examiner](https://www.socialmediaexaminer.com/feed/) | Social media marketing | RSS |

All six are free and need no API key, so the workflow needs **no credentials**.

## The unified schema

Every record, from every source, ends up with these 16 columns:

| Column | Meaning |
|---|---|
| `record_id` | Stable ID with a source prefix (`AD-`, `CMP-`, `MT-`, `HS-` …) |
| `source_name` / `source_type` | Where it came from, and what kind of record it is (`ad_copy`, `campaign_metadata`, `industry_article`) |
| `title_or_headline` | Headline, article title or campaign summary |
| `body_text` | Main text, HTML stripped, articles capped at 500 characters |
| `cta` | Call to action (ads only) |
| `brand_or_product` | Brand, company or product |
| `channel` / `campaign_type` / `audience` | Campaign context |
| `tags` | Hashtags or feed categories |
| `published_date` | Always `YYYY-MM-DD` |
| `source_url` | Link back to the original |
| `collected_at` | When this run fetched it |
| `is_complete` / `quality_flags` | Whether every required field for this source type is filled, and if not, what's missing |

## Quality checks

1. **Trim:** whitespace is collapsed in every text field.
2. **Dates:** every source's date format becomes `YYYY-MM-DD`. Unparseable or implausible dates are dropped and counted.
3. **Essentials:** a record without a title, date or source is dropped and counted.
4. **Duplicates:** articles are deduped by URL, campaigns by ID, and ads by product. The ad dataset repeats products with shuffled sentences.
5. **Completeness:** each source type has its own required fields. Missing ones are **flagged, not hidden**, e.g. `missing_cta`.
6. **Source health:** any source that fails or returns nothing is named in the report as `Source unavailable: …`, and the run carries on with the rest.

Every HTTP and RSS node retries 3 times, 2 seconds apart, and continues on error. One dead source never stops the run.

## Run it yourself

1. In n8n, go to **Workflows → Import from File** and choose [`workflow/one-pipe.json`](workflow/one-pipe.json).
2. Click **Execute workflow** to try it once. The CSV and report appear in the output of the last nodes.
3. **Activate** the workflow to run it every 3 days.

**Output location:** files are written to `/home/node/.n8n-files/`, the default writable folder in the n8n Docker image. On another setup, change the path in **Save CSV to Disk** and **Save Report to Disk**.

**Version note:** built and tested on a self-hosted n8n with Schedule Trigger 1.3, HTTP Request 4.2, RSS Read 1.2, Merge 3.2 and Code 2.

## n8n template version

[`workflow/one-pipe-n8n-template.json`](workflow/one-pipe-n8n-template.json) is the shareable version submitted to the n8n Creator Portal as *Combine RSS feeds and Hugging Face datasets into a quality-checked CSV*:

- Dataset names, the row window and feed URLs live in one **Set Source Configuration** node, so they can be changed without editing code.
- The write-to-disk nodes are removed, since n8n Cloud can't write files. Download the CSV and report from the **Export to CSV File** and **Export Report to JSON File** nodes, or connect a Google Drive or Sheets node after them.
- Every node is renamed to describe its step, and sticky notes explain each stage and the setup.

Import it the same way as the main workflow. It needs no credentials.

## Point it at a different source

Only stage ① and the matching Map node change. Stages ③ and ④ stay as they are.

1. Replace or add a fetch node: **HTTP Request** for JSON APIs, **RSS Read** for feeds.
2. Write a small **Map** Code node that turns the response into the 16 columns. Copy an existing one; they all follow the same pattern, including the "Source unavailable" check.
3. Connect it to **Merge All Sources**, raising its number of inputs if you added a source.
4. Add the new `source_type` and its required fields to `REQUIRED` in **Quality Checks & Cleaning**.

Ideas that fit the same shape:

| Use case | Free sources |
|---|---|
| Competitor monitoring | Google News RSS for brand names, Hacker News search ([Algolia API](https://hn.algolia.com/api)) |
| Research digest | [arXiv API](https://info.arxiv.org/help/api/index.html), [PubMed E-utilities](https://www.ncbi.nlm.nih.gov/books/NBK25501/) |
| Hiring signals | Public Greenhouse, Ashby and Lever job-board APIs |
| Customer feedback | App Store customer-review RSS feeds |
| AI datasets | Any Hugging Face dataset: change `dataset` in the request |

## Known limitations

- **Static datasets:** both Hugging Face requests fetch the first 100 rows (`offset=0`), so they return the same rows every run. Only the RSS feeds bring new data each time. Rotate `offset` if you want fresh rows.
- **Ad dates:** ad records carry the dataset's release date (2023-09-25), because the rows have no date of their own. They're flagged `date_from_dataset_release`.
- **Likely synthetic data:** the campaign dataset looks synthetic. It's useful for testing the pipeline's shape, not as real benchmark data.
- **Overwritten output:** each run overwrites the previous CSV and report instead of keeping a history.
