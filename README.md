# Lens⁸¹

**Know whether a Google Scholar result is a Research Paper or a Review Paper, directly in the search results.**

During my academic journey, I noticed that many of my batchmates and other early-career researchers struggled to tell research papers and review papers apart while browsing Google Scholar. Both often look similar at a glance, especially when you're scanning a page full of results.

Lens⁸¹ looks up each paper's abstract and adds a small badge next to the title, showing what the paper is most likely to be. This helps you spend more time reading the right papers instead of guessing.

---

## Features

* **Research vs. Review badges**, shown directly on the Google Scholar results page, with no extra clicks or tabs.
* **Instant + confirmed**. A quick title-based prediction appears immediately, then updates with an abstract-verified result a moment later.
* **Transparent explanations.** Click any badge to view the overall reasoning, along with each AI model's prediction, confidence score, and explanation. Nothing is hidden behind a black box.
* **Ensemble of up to 5 models.** Configure up to five OpenRouter API keys, each with its own model. Every model classifies the paper in parallel, and their probabilities are averaged into a single, more reliable result.
* **Cost-friendly fallback keys.** Mark a key as "Only use if needed" to keep it as a backup. It is only used, and only billed, if your primary model(s) fail.
* **Bring your own API key.** Data is only sent to OpenRouter (for classification) and Semantic Scholar/OpenAlex (for abstract lookup). There is no middleman server and no telemetry.
* **Local caching.** Results are cached for 30 days, so revisiting the same search does not trigger another API call.
* **Live per-tab counter.** View a running Research/Review count for the current tab from the toolbar icon.
* **Works without AI configuration.** If no API key is configured, Lens⁸¹ falls back to a lower-confidence keyword-based prediction instead of leaving papers unlabeled.

---

## How It Works

```text
Google Scholar page
   │  Content script reads each result's title
   ▼
Semantic Scholar / OpenAlex (queried in parallel)
   │  Looks up the title to retrieve the paper's abstract
   ▼
Your configured OpenRouter model(s) (1-5, run in parallel)
   │  Each returns a prediction, probability, and one-line explanation
   ▼
Ensemble averages the probabilities from every responding model
   ▼
Badge displayed next to the title:
📘 Research · 91%   or   📙 Review · 84%
   │  Click the badge
   ▼
"Why" panel showing the overall reasoning and every model's individual response
```

If an abstract cannot be found, Lens⁸¹ asks the AI to make its best judgment using only the title, and clearly indicates this in the badge.

If no AI model is configured, Lens⁸¹ falls back to a quick keyword-based prediction instead of leaving the result unlabeled.

---

## Why It Matters

Finding the right type of paper is one of the first steps in an effective literature review.

* **Research papers** present original experiments, methods, and findings.
* **Review papers** summarize and analyze existing research in a field.

Knowing the difference at a glance helps students learn more efficiently and helps researchers build stronger literature reviews.

---

## Installation

1. Clone this repository (or download and extract it).
2. Open `chrome://extensions`.
3. Enable **Developer mode**.
4. Click **Load unpacked** and select the project folder.
5. Click the Lens⁸¹ toolbar icon and open **Settings**.

---

## Configuration

The settings page contains five model slots. **Model 1 is required**, while Models 2-5 are optional. One model is enough to get started, but adding more models improves reliability by averaging their predictions.

For each model you configure:

1. Create an API key at https://openrouter.ai/keys and paste it into the extension. Keys are stored only in your browser and are sent only to `openrouter.ai`.
2. Choose a model slug from https://openrouter.ai/models (for example, `anthropic/claude-haiku-4.5` or `openai/gpt-4o-mini`) and paste it exactly as shown. Since available models and pricing change over time, always check the website instead of relying on hardcoded defaults.
3. Click **Test**. This sends one small request through OpenRouter so invalid API keys or incorrect model names are detected immediately.
4. Optionally enable **"Only use this key if the others fail."** This keeps the key as a backup and only uses it if every primary model fails or times out.

Click **Save**, then open any search page on `scholar.google.com`. Badges will appear automatically beside each result.

> Without an API key, Lens⁸¹ still works by using a lower-confidence keyword-based prediction instead of AI classification.

---

## Reliability

Lens⁸¹ includes several safeguards so slow networks or interrupted requests never leave the page looking broken.

* Every network request has its own timeout, and the complete classification pipeline is limited to 20 seconds. If a request takes too long, Lens⁸¹ falls back to the keyword-based prediction.
* If the browser's background worker never responds, the page stops waiting after 25 seconds instead of leaving a loading badge indefinitely.
* Closing a tab while classification is in progress is handled cleanly without console errors.

---

## Future Plans

* Support additional paper categories (Survey, Meta-analysis, Systematic Review)
* Confidence score tuning and calibration
* Paper summarization
* Citation insights
* Export reading lists
* Firefox support
* Dark mode improvements

---

## Contributing

Suggestions, feature requests, and pull requests are always welcome.

---

## Disclaimer

Classification is AI-assisted and may not always be correct. It is designed to help you quickly understand what you're looking at, not to replace reading the paper itself. Please verify classifications whenever accuracy is important.

---

## Version History

**V1**
Original release. Single OpenRouter API key with one model, abstract lookup using Semantic Scholar/OpenAlex, instant title-based predictions, per-tab counter, and local caching.

**V2**
Introduced the multi-model ensemble. Up to five API keys, each with its own model, classify every paper in parallel. Results are combined by averaging Research and Review probabilities.

**V3**
Added explainability and reliability improvements. Clicking a badge now shows the complete reasoning, including every model's individual response. Introduced cost-saving fallback keys and improved handling of slow networks, stalled requests, and tabs closing during classification.

---

Built with the goal of making academic literature exploration a little easier for every student and early-career researcher.


*Sarthak Turkane* <3
