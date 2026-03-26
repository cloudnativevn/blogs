---
layout: page
title: Contribute Guide
date: 2026-03-26
permalink: /writing-hub/contribute-guide/
---

<section class="guide-intro">
  <p class="guide-kicker">Writing Hub</p>
  <h2>Submit Your Post to GitHub</h2>
  <p>
    Follow this guide to submit your article to the Cloud Native Vietnam blog. Keep the event
    board open while preparing your draft so your topic and status stay in sync.
  </p>
  <a class="hub-btn hub-btn--solid" href="https://github.com/cloudnativevn/blogs" target="_blank" rel="noopener">Open CNCF VN GitHub</a>
</section>

## 1) Repository Setup

1. Fork the repository to your GitHub account.
2. Clone your fork to local machine.
3. Create a branch for your article, for example: `feat/writing-hub-service-mesh`.

## 2) Create Your Article File

1. Add a new markdown file inside `_articles/`.
2. Use a clear file name with lowercase words and hyphens, for example: `service-mesh-in-production.md`.
3. Copy this front matter template and update values:

```yaml
---
layout: post
title: "Your Article Title"
date: 2026-03-26
author: "Your Name"
tags:
  - kubernetes
  - writing-hub
excerpt: "One or two sentence summary for readers."
---
```

## Event Submission Note (Optional)

If you want to submit to a Writing Hub event, keep the same normal contribution flow above,
but place your article in the event folder instead of the top-level `_articles/` directory.

- Writing Contest 2026 folder: `_articles/writing-hub/writing-contest-2026/`
- Event source file used by organizers: `Master file_Writing contest.xlsx`

You can use the sample post in the event folder as a template and then rename it for your topic.

## 3) Write Content Guidelines

- Keep the article practical and experience-based.
- Add code snippets, diagrams, or command examples when useful.
- Prefer short sections with meaningful headings.
- Include references for external sources.
- End with key takeaways or lessons learned.

## 4) Validate Before You Open PR

- Run the site locally if possible to verify rendering.
- Check markdown formatting and links.
- Confirm front matter fields are complete.
- Ensure there are no sensitive credentials or private data.

## 5) Open Pull Request

1. Push your branch to your fork.
2. Open a Pull Request to this repository main branch.
3. Use PR title format: `Writing Hub: <your article title>`.
4. In the PR description, include:
   - Event board topic or row reference
   - Short summary of the article
   - Anything reviewers should focus on

## 6) Review and Publish

- Organizers may request edits for clarity, technical accuracy, or style.
- After approval and merge, the article will be published automatically.
- Update your row on the event board to the latest status.

<section class="guide-checklist">
  <h3>Quick Submission Checklist</h3>
  <ul>
    <li>Article file created in `_articles/`</li>
    <li>If submitting to an event, article file is created in the correct event folder</li>
    <li>Front matter filled correctly</li>
    <li>Content proofread and links verified</li>
    <li>PR opened with proper title and summary</li>
  </ul>
</section>
