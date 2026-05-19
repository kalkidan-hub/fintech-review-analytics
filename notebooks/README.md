
# Web Scraping Notebook — Methodology & Notes

This notebook collects Google Play reviews for selected banking apps and performs basic cleaning and QA.

## Methodology

- Source: Google Play Store via the `google_play_scraper` Python package.
- Apps: configured in the notebook (`CBE_BANK_ID`, `BOA_BANK_ID`, `DASHEN_BANK_ID`).
- Fetch parameters: the notebook uses `lang` and `country` parameters (example: `lang='en'`, `country='et'`) and `sort=Sort.NEWEST` to fetch recent reviews. The `count` parameter controls how many reviews to request per call; the notebook fetches batches and may loop across calls using continuation tokens.
- Per-app flow:
	1. Fetch reviews for the app and append to a raw list.
	2. Build a `DataFrame` (`df_raw_<app>`) from the raw results.
	3. Apply cleaning steps: drop rows with critical missing fields, deduplicate, normalize/parse `date`, validate `rating`, detect and drop non-English reviews, collapse whitespace and strip text.
	4. Produce cleaned `DataFrame` (`df_<app>_cleaned`) and save to `data/processed/<app>_bank_reviews_clean.csv`.

## Date range used

- The notebook fetches the most recent reviews (using `Sort.NEWEST`) at the time the notebook is executed. There is no explicit absolute start/end date parameter in the scraper calls — instead the dataset reflects the reviews available at fetch time.
- If you need a specific date range, modify the fetching loop to filter by the review `date` field after parsing (e.g., `df[df['date'].between('2026-01-01', '2026-05-15')]`).

## Key cleaning & validation steps

- Drop rows with missing critical values (e.g., missing `review_id`, `review`, `date`).
- Deduplicate exact rows and by `review_id`.
- Normalize `date` to `YYYY-MM-DD` strings using `pd.to_datetime(...).dt.strftime('%Y-%m-%d')`.
- Validate `rating` values and cast to `int` (keep values between 1 and 5).
- Remove non-English reviews using a language-detection fallback (the notebook uses `langdetect` when available, otherwise falls back to an ASCII heuristic).
- Trim and collapse whitespace in review text; compute review-length statistics and generate a per-bank report.

## Limitations & caveats

- Locale vs. language: `lang` (e.g., `'en'`) and `country` (e.g., `'us'`, `'et'`) control the storefront and locale used for the request, but they do NOT guarantee that each review's text is in that language. The notebook therefore performs explicit language detection to filter non-English text.
- Market coverage: there is no single Google Play call that returns all markets at once. To collect reviews across markets, you must iterate over desired `country` codes and merge results — this can create duplicate reviews across markets.
- Pagination and continuation tokens: the scraper uses continuation tokens to page through results. Tokens can expire or behave inconsistently; handle exceptions and be prepared to resume or restart fetching.
- Sampling vs. complete history: the notebook requests recent reviews (`Sort.NEWEST`) and a configured `count`; depending on the app's review volume and the `count` limits, you may not retrieve the app's full historical review set.
- Language detection accuracy: `langdetect` is good for short texts but not perfect; short reviews or mixed-language reviews can be misclassified. The notebook falls back to a simple ASCII check when `langdetect` fails.
- Rate limits and blocking: aggressive scraping may hit Google Play rate limits or trigger anti-bot measures. Add delays between requests and respect terms of service.
- Time zones and `datetime64` units: review `date` values are normalized to a date string. When keeping `datetime64` objects, be aware of the unit (e.g., `datetime64[ns]` vs `datetime64[us]`) and timezone-naive values.

## Reproducibility

1. Activate the project virtual environment and install dependencies:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1  # PowerShell
pip install -r requirements.txt
pip install langdetect nbqa black  # optional helpers for language check and formatting
```

2. Open `notebooks/web_scraping.ipynb`, run cells in order. Exported CSVs are placed under `data/processed/`.



## Sentiment analysis (next steps)

Planned pipeline to add sentiment analysis to the cleaned reviews:

1. Labeling & sample size
	- Inspect a stratified sample of cleaned reviews per bank and label a small validation set (e.g., 500–2,000 examples) for positive/neutral/negative.

2. Baseline models
	- Start with lightweight rule-based baselines (VADER or TextBlob) for English text to get quick signals.
	- Evaluate baseline accuracy on the labeled validation set.

3. Transformer-based model
	- If higher accuracy is needed, fine-tune a pre-trained transformer (e.g., `distilbert-base-uncased` or a domain-adapted model) on the labeled data.
	- Use `transformers` + `datasets` for training and evaluation; consider multilingual models if you expand beyond English.

4. Handling language and short texts
	- Run sentiment only on rows that pass the English filter.
	- For very short reviews, consider aggregating at the user/app level or using ensemble heuristics.

5. Metrics and validation
	- Evaluate with accuracy, F1 (macro), precision/recall per class, and confusion matrices.
	- Track per-bank performance to detect dataset drift.

6. Productionizing
	- Add a notebook cell or script to compute sentiment labels and store them as a new column (`sentiment`, `sentiment_score`) in the cleaned CSVs.
	- Save model artifacts, a reproducible training script, and inference wrapper.

7. Compute & cost
	- Baselines run locally; transformer fine-tuning benefits from GPU. Estimate compute before training large models.

## Sentiment & Thematic Analysis — Work completed

Summary of what was executed in this project so far (notebooks: `sentiment_analysis.ipynb`, `thematic_analysis.ipynb`):

- Preprocessing: implemented a small NLP pipeline that cleans text, tokenizes, removes stopwords and lemmatizes using NLTK. Notebook ensures required NLTK data are available (`punkt_tab`, `wordnet`, `stopwords`).

- Sentiment labeling: used HuggingFace `transformers` pipeline with `distilbert-base-uncased-finetuned-sst-2-english` to produce `sentiment_label` and `sentiment_score` for each review (stored in `banks_sentiment.csv`). VADER/TextBlob are available as quick baselines in the notebook for comparison.

- Thematic extraction:
	- TF‑IDF n‑grams (unigrams + bigrams) to surface high‑importance phrases.
	- spaCy noun‑chunk extraction to capture readable phrase candidates (requires the `en_core_web_sm` model).
	- Semantic phrase clustering: used `sentence-transformers` (`all-MiniLM-L6-v2`) to embed candidate phrases and KMeans to group related phrases into 3–5 clusters per bank. Representative phrase per cluster is produced.
	- Topic modeling: added LDA (sklearn) to discover latent topics and attached `lda_topic` and `lda_topic_confidence` to the dataframe.

- Outputs & visualization:
	- Per‑bank theme clusters assigned to reviews as `theme_cluster` and readable `theme_label`.
	- Wordclouds generated per bank+cluster (saved to `outputs/wordclouds/`).
	- Aggregation tables computed: `bank_summary`, `rating_summary`, `bank_rating_summary` (notebook cells show examples of plotting these summaries).

- Saved artifacts:
	- `data/processed/banks_sentiment.csv` — intermediate sentiment-labeled data.
	- `data/processed/banks_sentiment_with_themes.csv` — final dataset with theme labels and LDA topics.

How to reproduce
- Make sure the project virtual environment is activated and dependencies installed (see `requirements.txt`). If you need CPU-only PyTorch, add the extra index and `torch` as discussed in the project root.
- Run `notebooks/sentiment_analysis.ipynb` then `notebooks/thematic_analysis.ipynb` in order.

Notes
- Some steps (transformer inference, sentence‑transformers encoding) may be slow on CPU; consider using a GPU environment if available.
- Thematic cluster names are initially auto-generated from representative phrases — review and rename them to human-friendly names if producing a final report.



