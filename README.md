# Lens⁸¹

A lightweight Chrome extension that helps you tell whether a paper on Google Scholar is a **Research Paper** or a **Review Paper** - right in the search results, using AI-powered abstract classification.

## Why I Built This

During my academic journey, I noticed that a lot of my batchmates and researchers who were just starting out struggled to tell research papers and review papers apart while browsing Google Scholar.

That confusion is common — both types of papers can look similar at first glance, especially when you're quickly scanning a page of search results.

To make that easier, I built **Lens⁸¹**. It looks up a paper's abstract and displays a small badge showing whether it's most likely a Research Paper or a Review Paper, so you can save time and make better calls about what to read.

## Features

- Detects Research vs. Review papers, right on the Google Scholar results page
- An instant title-based guess appears immediately, then upgrades to an abstract-confirmed badge
- A running Research/Review count for the current tab, visible from the toolbar icon and popup
- Results are cached locally, so revisiting a search doesn't cost extra API calls
- Bring-your-own API key — nothing is sent anywhere except OpenRouter and the abstract lookup services
- Simple, unobtrusive badge interface — no changes to how you already use Scholar

## How It Works

```

Google Scholar page
   │  the extension reads each result's title
   ▼
Semantic Scholar / OpenAlex
   │  looked up by title to retrieve the paper's abstract
   ▼
AI model (via OpenRouter, your own key)
   │  classifies the abstract
   ▼
Badge shown next to the title:
   📘 Research Paper   or   📙 Review Paper
   
```

Google Scholar doesn't expose abstracts directly, so Lens⁸¹ looks them up from Semantic Scholar (falling back to OpenAlex) before classifying. If no abstract can be found, it falls back to a quick title-keyword guess instead of leaving the result unlabeled.

## Why It Matters

Finding the right type of paper is an important first step in any literature review.

- **Research papers** present original experiments, methods, and findings.
- **Review papers** summarize and analyze existing research in a field.

Knowing the difference at a glance helps students learn more efficiently and helps researchers build stronger literature reviews.

## Installation

1. Clone this repository.
2. Open `chrome://extensions`.
3. Enable **Developer mode** (top right).
4. Click **Load unpacked** and select the project folder.
5. Click the Lens⁸¹ icon in your toolbar → **Open settings**.
6. Add your [OpenRouter](https://openrouter.ai/keys) API key and a model slug from [openrouter.ai/models](https://openrouter.ai/models), then hit **Test connection**.
7. Open Google Scholar and start browsing — badges appear next to each result automatically.

> Without an API key configured, Lens⁸¹ still works, but falls back to a lower-confidence title-keyword guess instead of an abstract-based classification.

## Future Plans

- Support for more paper categories (Survey, Meta-analysis, Systematic Review)
- Confidence score tuning and calibration
- Paper summarization
- Citation insights
- Export reading lists
- Firefox support
- Dark mode improvements

## Contributing

Suggestions, feature requests, and pull requests are welcome.

## Disclaimer

Classification is AI-assisted and may not always be correct. It's meant to help you quickly get a sense of what you're looking at - not to replace actually reading the paper. Please verify when accuracy matters.

---

Built with the goal of making academic literature exploration a little easier for every student and early-career researcher.

*Sarthak Turkane* <3
