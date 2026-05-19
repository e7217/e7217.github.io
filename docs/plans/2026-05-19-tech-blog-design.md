# Development Blog Design

Date: 2026-05-19

## Goal

Build a personal technical blog for broad development records, with Smart Factory
and AI as the main themes. The site should stay simple at first, make posting easy,
and let project records naturally become portfolio material over time.

## Positioning

The blog is primarily a development journal, not a polished hiring portfolio.
It should emphasize:

- Smart Factory experiments and manufacturing automation records
- AI, ML, LLM, and automation experiments
- Personal project design, implementation, troubleshooting, and operation notes
- Practical engineering decisions across backend, frontend, data, and deployment

## Initial Information Architecture

Use a small number of broad sections first. Split them later only when enough posts
exist to justify more structure.

```text
Home
├─ Recent posts
├─ Main Smart Factory / AI records
├─ Active or representative projects
└─ Short profile and links

Blog
├─ Project Log
├─ Smart Factory
├─ AI
├─ Engineering
└─ Troubleshooting

Projects
├─ Personal project summaries
├─ Related development-log links
└─ Demo, repository, screenshots, and status where available

About
└─ Interests, working themes, tech stack, GitHub, and contact links
```

## Category Strategy

Start with five broad categories:

- `Project Log`: personal project progress, design notes, release notes, retrospectives
- `Smart Factory`: manufacturing automation, MES, equipment monitoring, process data
- `AI`: ML, LLM, RAG, model usage, AI-assisted automation
- `Engineering`: backend, frontend, database, infrastructure, general development
- `Troubleshooting`: error analysis, debugging, deployment issues, operational fixes

Use tags for details instead of creating many categories too early.

Example:

```yaml
categories: [AI]
tags: [llm, rag, automation, python]

categories: [Smart Factory]
tags: [mes, plc, opc-ua, monitoring]

categories: [Engineering]
tags: [spring, react, postgresql, docker]
```

Possible future splits:

- `AI` -> `LLM`, `Computer Vision`, `MLOps`
- `Smart Factory` -> `MES`, `Equipment Monitoring`, `Data Collection`, `Automation`
- `Engineering` -> `Backend`, `Frontend`, `Database`, `DevOps`

## Post Template

Project and engineering posts should follow a consistent structure when useful:

```markdown
# Title

## Background
Why this work was needed.

## Goal
What this post tries to implement, verify, or explain.

## Design
System structure, data flow, technology choices, and trade-offs.

## Implementation
Core implementation details and code notes.

## Problems And Fixes
What failed, why it failed, and how it was resolved.

## Result
Behavior, screenshots, measurements, limitations, or remaining issues.

## Next Steps
Follow-up improvements or related posts.
```

The template is a guide, not a rule. Short troubleshooting posts can be much simpler.

## Technical Direction

Use GitHub Pages with Jekyll and the existing Minimal Mistakes theme configuration.
This fits the current repository, keeps writing Markdown-based, and avoids adding
unnecessary frontend tooling before the blog has enough content to require it.

The first implementation should focus on:

- Replacing the placeholder `index.html`
- Updating `_config.yml` site metadata and navigation
- Adding blog, projects, and about pages
- Adding an initial `_posts` structure and example post
- Keeping styling minimal and readable

## Success Criteria

- The first screen clearly communicates a Smart Factory / AI centered technical blog.
- New posts can be added with Markdown only.
- The top-level category structure stays small and easy to maintain.
- Project posts can be linked from both blog and project pages.
- The site remains compatible with GitHub Pages deployment.
