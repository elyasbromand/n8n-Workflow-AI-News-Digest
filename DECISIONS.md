# Build Decisions & Rationale

This doc explains the _why_ behind each non-obvious choice in the workflow — useful if you're forking this and wondering why something isn't built the "obvious" way.

## Why no "split into articles" step

RSS Read nodes already output one item per article. Merge (append) already gives a flat list. A split step would be a no-op.

## Why no "fetch full article content" step

RSS `contentSnippet` fields already carry enough summary text for the LLM to work from. Fetching full pages adds fragility (paywalls, anti-bot, HTML parsing) for marginal gain. Revisit only if snippet quality degrades.

## Why one LLM call instead of "summarize each → aggregate → build digest"

Sending the whole article list in one prompt lets the LLM **rank importance across all sources at once** — critical to the actual goal ("top news," not "every news item"). Per-article summarization has no mechanism for cross-source importance ranking and burns far more free-tier API calls.

## Why Anthropic uses an RSSHub mirror, not the official site

Anthropic has no official RSS feed. Two community options exist:

- `raw.githubusercontent.com/taobojlen/anthropic-rss-feed` — well-formed XML but `description` == `title` (no real summary content, useless for the LLM step)
- `rsshub.bestblogs.dev/anthropic/news` — **chosen**, has real body content in the description field, matching what other sources provide

"Continue on Fail" is enabled on this node since it depends on a third-party RSSHub instance staying up. If it goes down, the workflow still runs fine off the other 5 sources.

## Why Hugging Face uses the "Daily Papers" feed, not the official blog feed

HF's official blog feed (`huggingface.co/blog/feed.xml`) has no description field at all — title only. Swapped to:

```
https://raw.githubusercontent.com/huangboming/huggingface-daily-paper-feed/refs/heads/main/feed.xml
```

This tracks trending papers on HF (title, authors, summary, upvotes) — arguably better aligned with "important AI news" than generic engineering blog posts anyway. Also has "Continue on Fail" enabled (same single-point-of-failure reasoning).

## Why the Filter uses a keyword regex, not stricter logic

GitHub Blog and NVIDIA Developer post lots of non-AI content (Git tutorials, GPU drivers). The regex is intentionally **broad/loose** rather than clever — a borderline false-positive slipping through is safer than false-negatives killing something important. The LLM's "pick top 5-8" step acts as the real importance filter; this step only removes obvious noise.

Current keyword list (expand as needed):

```
ai|llm|gpt|claude|gemini|neural|machine learning|deep learning|
transformer|generative|agent|opus|sonnet|haiku
```

## Why "Remove Items Seen Before" instead of the default duplicate-detection mode

The default Remove Duplicates mode only catches duplicates _within a single run_ (e.g. two feeds reporting the same story same day). RSS feeds always carry several weeks of backlog by design, so without persistent memory, the same article would resurface every morning until it naturally aged out of the feeds. "Remove Items Seen Before," keyed on `link`, remembers everything ever sent across all past executions — for free, no external DB.

**Gotcha:** this history lives on this specific node instance. Deleting/recreating this node resets it — expect an "everything looks new again" run if that happens.
