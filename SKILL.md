---
name: url2zh
description: "Translate a non-Chinese web page into Chinese using a 3-step translation method (literal → identify issues → liberal). Usage: /url2zh <url>. Fetches the URL, checks it's non-Chinese content, translates with high quality, and saves the result as a markdown file."
user_invocable: true
---

# URL to Chinese

You are a professional translator. Your job is to translate a web page from its original language into Chinese, producing a high-quality markdown document.

## Input

The user provides a URL as the command argument:

```
/url2zh <url>
```

Argument source: $ARGUMENTS

If no URL is provided, ask the user for one.

## Step 1: Fetch and Validate

1. Use the WebFetch tool to fetch the URL content.
2. **Language check**: Determine if the page content is primarily in Chinese. If the main body text is already in Chinese, inform the user: "该网页内容已经是中文，无需翻译。" and **stop**.
3. If it is non-Chinese, proceed to translation.

## Step 2: Extract Key Information

From the fetched content, extract:

- **Title** of the article/page
- **Author** (if available)
- **Publication date** (if available)
- **Main body content** (the article text, code blocks, lists, etc.)
- **Images**: Collect all image URLs from the content. Preserve them as-is using absolute URLs (ensure they are complete URLs, not relative paths). If an image URL is relative, prepend the source domain to make it absolute.

## Step 3: Three-Step Translation

Apply the following 3-step translation method. You MUST internally perform all three steps, but only output the final result (Step 3).

### Translation Rules

- Accurately convey the facts and context of the original text
- Retain specific English terms or proper nouns (e.g., DALL-E, GPT, BERT)
- Retain company/product names and abbreviations in English
- Retain cited paper/article titles in English
- People's names are NOT translated — keep the original form
- For technical terms, on first occurrence include the English original in parentheses, e.g., "大语言模型（LLM）"
- Convert full-width brackets `（）` to half-width `()` when containing English
- Preserve all original markdown formatting (headings, lists, code blocks, bold, italic, links)
- Preserve all images using their original absolute URLs

### Step 3a: Literal Translation (internal, do not output)

Translate the content literally, preserving the original structure and format completely. Do not omit any information.

### Step 3b: Identify Issues (internal, do not output)

Review the literal translation and identify specific problems:
- Expressions that don't follow natural Chinese conventions
- Sentences that are not smooth or fluent
- Parts that are difficult to understand

### Step 3c: Liberal Translation (this is the final output)

Based on the literal translation and the identified issues, produce the final translation that:
- Faithfully conveys the original meaning
- Reads naturally and fluently in Chinese
- Is easy to understand for a Chinese reader
- Maintains the exact same structure and formatting as the original

## Step 4: Write Output

### Output Format

Write the final translation as a markdown file with this structure:

```markdown
# [Translated Title]

> 原文: [original URL]
> 作者: [Author if available]
> 日期: [Date if available]

---

[Translated content with all original formatting, images, code blocks preserved]
```

### File Naming

Generate the filename as: `{slug}.md`

Where `{slug}` is derived from the article title (or URL path if no title):
- Use lowercase
- Replace spaces and special characters with hyphens
- Keep it concise (max ~60 chars)
- Use Chinese pinyin or English keywords from the title

Example: `levels-of-agentic-engineering.md`, `understanding-transformer-architecture.md`

### File Location

- Primary: write to `~/Repos/ai/url2zh/{slug}.md`
- If that directory does not exist, write to the current working directory instead

## Step 5: Commit and Push

After writing the file to `~/Repos/ai/url2zh/`:

1. `cd ~/Repos/ai/url2zh`
2. `git add {slug}.md`
3. Commit with message: `translate: {article title in English or short description}`
4. `git push`

Skip this step if the file was written to the current working directory instead (i.e., the primary directory did not exist).

### Important

- Ensure all images use absolute URLs so they render correctly
- Preserve code blocks with their original language annotations
- Keep the markdown well-structured and readable
- Do NOT include the intermediate translation steps (literal translation, issue identification) in the output file — only the final liberal translation
