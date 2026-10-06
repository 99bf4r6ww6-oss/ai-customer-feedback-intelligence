# AI Customer Feedback Intelligence

An end-to-end AI analytics project that transforms unstructured e-commerce customer reviews into structured, decision-ready customer intelligence using Python, the OpenAI API, and validated structured outputs.

The project combines traditional descriptive analytics with LLM-based text classification to identify customer sentiment, product issues, customer intent, urgency, and recommended business actions.

> **Project status:** Analysis in progress. The full dataset contains 22,641 usable customer reviews. A stratified 500-review sample is being enriched with LLM-generated structured classifications. Batch inference uses checkpoint recovery to preserve completed results during API interruptions and rate limits.

---

## Business Problem

Customer reviews contain valuable information about product quality, fit, design, customer satisfaction, and purchase intent, but most of this information exists as unstructured text.

Manually reviewing thousands of comments is slow and difficult to scale.

This project asks:

> **How can a retailer transform unstructured customer reviews into structured customer intelligence for faster issue identification and business decision-making?**

The analytical workflow converts free-text reviews into standardized business dimensions that can be aggregated, validated, and visualized.

---

## Project Workflow

```text
Raw Customer Reviews
        ↓
Data Quality Audit
        ↓
Data Cleaning
        ↓
Baseline Customer Analytics
        ↓
Stratified LLM Sample
        ↓
OpenAI API Classification
        ↓
Pydantic Structured Outputs
        ↓
Validation & Quality Checks
        ↓
Customer Issue Analytics
        ↓
Executive Dashboard
        ↓
Business Recommendations
```

---

## Dataset

**Source:** Women's E-Commerce Clothing Reviews dataset from Kaggle.

The original dataset contains **23,486 customer reviews** and 11 fields, including:

- Review Text
- Rating
- Recommended IND
- Age
- Clothing ID
- Division
- Department
- Product Class
- Positive Feedback Count

### Data Quality Audit

| Metric | Result |
|---|---:|
| Raw records | 23,486 |
| Usable text reviews | 22,641 |
| Missing review text | 845 |
| Duplicate rows | 0 |
| Average rating | 4.18 / 5 |
| Recommendation rate | 81.9% |
| 1–2 star reviews | 2,370 |
| Low-rating rate | 10.5% |

Rows without usable `Review Text` were removed from text analysis while the original rating and recommendation variables were preserved for downstream validation.

---

## Baseline Customer Analytics

Before introducing an LLM, the project establishes a conventional analytical baseline from the full set of **22,641 usable reviews**.

### Rating Distribution

| Rating | Reviews |
|---:|---:|
| 1 star | 821 |
| 2 stars | 1,549 |
| 3 stars | 2,823 |
| 4 stars | 4,908 |
| 5 stars | 12,540 |

The dataset is strongly skewed toward favorable customer experiences, making it important to preserve lower-rated feedback when constructing the LLM analysis sample.

### Department-Level Feedback

| Department | Reviews | Average Rating |
|---|---:|---:|
| Bottoms | 3,662 | 4.28 |
| Intimate | 1,653 | 4.27 |
| Jackets | 1,002 | 4.25 |
| Tops | 10,048 | 4.16 |
| Dresses | 6,145 | 4.14 |
| Trend | 118 | 3.84 |

Although the Trend department has the lowest average rating, its sample size is only 118 reviews. Therefore, the project avoids treating it as the retailer's largest customer-experience problem based on average rating alone.

In absolute volume, **Tops and Dresses account for substantially more low-rated feedback**, with 1,096 and 681 1–2 star reviews respectively.

---

## Why Use an LLM?

Star ratings identify whether customers are broadly satisfied, but they do not directly explain:

- **what** customers liked or disliked,
- whether the problem concerns fit, quality, design, comfort, or value,
- whether the customer is praising, complaining, recommending, or considering a return,
- how urgently a problem should be investigated,
- or what action a business team could take.

The LLM layer converts each review into a consistent structured record that can be analyzed like conventional tabular data.

---

## LLM Sampling Strategy

Running qualitative analysis only on randomly selected reviews would heavily favor 4- and 5-star feedback because of the dataset's rating imbalance.

To ensure that both positive and negative customer experiences are represented, the project constructs a **500-review stratified sample**:

- 100 × 1-star reviews
- 100 × 2-star reviews
- 100 × 3-star reviews
- 100 × 4-star reviews
- 100 × 5-star reviews

Sampling uses a fixed random seed for reproducibility.

### Important Methodological Limitation

Because the LLM sample deliberately contains equal numbers of reviews from each rating category, **AI-classification percentages from this sample should not be interpreted as population prevalence estimates**.

Full-dataset statistics are used for overall descriptive conclusions. The stratified LLM sample is used to investigate qualitative patterns and issue types across different customer experiences.

---

## Structured AI Classification

Each sampled review is submitted to the OpenAI API and converted into a validated Pydantic object.

The output schema contains five business dimensions:

### Sentiment

```text
Positive
Neutral
Negative
```

### Issue Category

```text
Fit & Sizing
Product Quality
Style & Design
Comfort
Price & Value
Other
```

### Customer Intent

```text
Praise
Complaint
Return
Recommendation
General Feedback
```

### Urgency

```text
Low
Medium
High
```

### Recommended Action

A short business-oriented action derived from the review.

Example structured output:

```json
{
  "sentiment": "Negative",
  "issue_category": "Style & Design",
  "intent": "Recommendation",
  "urgency": "Low",
  "recommended_action": "Re-evaluate side-panel placement and test the design across different body types."
}
```

---

## Structured Output Validation

The project uses **Pydantic and constrained categorical fields** rather than relying on unrestricted free-form model responses.

This ensures that downstream analysis receives standardized categories instead of inconsistent variations such as:

```text
Sizing
Size Problem
Fit Issue
Poor Fit
```

All of these concepts must instead conform to the predefined:

```text
Fit & Sizing
```

This makes LLM output directly usable in a pandas analytical workflow.

---

## Initial Quality Check

Before batch inference, reviews representing different rating levels were manually inspected.

Examples included:

- a 1-star, non-recommended review classified as **Negative / Style & Design / Recommendation / Low urgency**
- a 3-star, non-recommended review classified as **Negative / Product Quality / Return / Medium urgency**
- a 5-star, recommended review classified as **Positive / Style & Design / Praise / Low urgency**

This initial test demonstrated that the classifier does not simply convert star ratings into sentiment labels. For example, a 3-star review can still be classified as negative when the underlying text describes a meaningful product problem.

These checks are exploratory sanity checks rather than formal model-accuracy estimates.

---

## Fault-Tolerant Batch Inference

The LLM enrichment stage is implemented as a recoverable batch-processing pipeline.

The pipeline includes:

- sequential API processing,
- structured Pydantic validation,
- token-usage tracking,
- periodic checkpoint persistence,
- exception handling,
- API rate-limit detection,
- and restart-safe processing using completed review IDs.

Checkpoint results are written periodically to:

```text
ai_feedback_checkpoint.csv
```

If inference is interrupted, previously completed reviews are loaded from the checkpoint and skipped rather than being submitted to the API again.

This prevents duplicate API usage and makes the workflow resilient to temporary API or network interruptions.

---

## Validation Strategy

The completed project will evaluate LLM output using several complementary checks.

### Rating-Based Sanity Check

Ratings provide an imperfect but useful external reference:

```text
1–2 stars → expected predominantly negative
3 stars   → mixed / neutral / negative depending on text
4–5 stars → expected predominantly positive
```

Agreement with this mapping will be reported as a **sanity-check agreement**, not as model accuracy, because star ratings are not ground-truth sentiment labels.

### Recommendation Consistency

AI sentiment and intent will also be compared with the original `Recommended IND` variable.

### Human Review

A random subset of structured outputs will be manually inspected for semantic reasonableness before final conclusions are reported.

---

## Planned Customer Intelligence Analysis

After LLM enrichment is complete, the project will analyze:

- sentiment distribution,
- dominant customer issue categories,
- negative-review drivers,
- complaint and return intent,
- urgency distribution,
- issue patterns by department,
- relationship between AI sentiment and star rating,
- relationship between AI sentiment and recommendation behavior,
- and recurring recommended business actions.

Results will distinguish clearly between:

**full-dataset descriptive statistics** and **patterns observed within the stratified LLM sample**.

---

## Executive Dashboard

The final project will include an executive-style visualization summarizing:

- overall review KPIs,
- rating distribution,
- low-rating volume by department,
- AI-classified customer issue categories,
- sentiment and urgency patterns,
- and key business insights.

Dashboard development is performed in Python using Matplotlib.

---

## Technology Stack

**Data Analysis**
- Python
- pandas
- NumPy
- Jupyter Notebook

**AI / NLP**
- OpenAI API
- GPT-5 mini
- Structured Outputs
- Pydantic

**Visualization**
- Matplotlib

**Engineering**
- JSON-compatible structured outputs
- API token tracking
- exception handling
- checkpoint recovery
- Git / GitHub

---

## Repository Structure

```text
ai-customer-feedback-intelligence/
│
├── README.md
├── ai_customer_feedback_intelligence.ipynb
├── customer_feedback_dashboard.png
└── ai_feedback_checkpoint.csv
```

> The original Kaggle dataset is not redistributed in this repository. See the dataset source for access.

---

## Current Status

- [x] Dataset acquisition
- [x] Data quality audit
- [x] Review-text cleaning
- [x] Baseline customer analytics
- [x] Stratified 500-review sample
- [x] Structured LLM taxonomy
- [x] OpenAI API integration
- [x] Pydantic output validation
- [x] Checkpoint/recovery pipeline
- [ ] Complete 500-review LLM enrichment
- [ ] Quantitative validation
- [ ] Customer issue analysis
- [ ] Executive dashboard
- [ ] Final business recommendations

---

## Key Analytical Principles

This project intentionally separates:

1. **Descriptive evidence** from the complete review dataset,
2. **LLM-derived qualitative patterns** from the stratified sample,
3. **validation proxies** from true model accuracy,
4. and **observed relationships** from causal conclusions.

The goal is not simply to classify customer reviews with an LLM, but to build a reproducible workflow that converts unstructured feedback into structured business intelligence while preserving analytical transparency.

---

## Author

**Zihan Liu**  
University of Toronto  
Economics & Mathematics | Statistics
