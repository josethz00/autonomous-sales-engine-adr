# Limitations and Edge Cases

With the current design, we may face some limitations for some edge cases. This document aims to address these limitations, explain them, and propose possible workarounds for them.

## What works? ✅

A client opens the Gallium platform, fills in basic details about its brand, and about its main competitors (optional). Then, it uploads files with past emails from its own company and the competitor, or connects via OAuth providers, so the system can extract sales and marketing data. After that, the ingestion begins, the blueprints are successfully generated with competitors' data extracted from inbox seeding, web scraping, and custom documents provided and the campaign is launched, bringing a huge value to the client's operation!

## What Doesn't Work? ❌

There are some scenarios where the system might struggle to produce good blueprints with the potential to generate great emails. Due to AI's nature, the blueprint will still be generated, however, it might not be of good quality. One realistic failure case happens when there is not enough data available, such as:

- NO INBOX SEEDING
- COMPETITORS' WEBSITE WITH LOW INFO
- COMPETITORS DON'T HAVE ANY PUBLIC BLOGS OR SOCIAL MEDIA FOR THE SYSTEM TO LEARN
- EMPTY OR NOT ENOUGH DATA IN THE MAILBOX

All these edge cases are around little data available or bad data, and it's important to establish some heuristics and metrics to warn the clients that, in the current state of the data, the outcome of the marketing and sales campaigns might be too generic. In this scenario, the system works for the simple flow, but fails to produce high-quality blueprints for more complex requests that require stronger competitive and behavioral signals.

A workaround for that is having an extra input for the user to specify extra details about its company and competitors, like websites (if any), strategies, social media (if any), and any additional context that he judges important for our AI to know. From an architectural perspective, this would require detecting low-data scenarios during ingestion and allowing the blueprint generation to incorporate this additional structured input as a fallback, which I already hinted [at here](https://github.com/josethz00/autonomous-sales-engine-adr/blob/main/DATABASE_INDEXING_AND_BLUEPRINT.md#campaigns) by suggesting the `additional_context`  field for the `campaigns` table.
