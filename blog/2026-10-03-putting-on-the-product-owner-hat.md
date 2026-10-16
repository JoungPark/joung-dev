---
slug: putting-on-the-product-owner-hat
title: Putting on the Product Owner Hat
authors: [joung]
tags: [second-memory]
---

I had the idea for Second-Memory, but it was still just an idea—too vague to start thinking about architecture or implementation. I wanted to develop it further before building, so I put on the **Product Owner hat** and followed the kind of process I had experienced in professional work.

{/* truncate */}

I started by defining the initiative: what I was trying to achieve and what I wanted the product to be. Then I wrote a PRD. Writing it made me think through the product in more detail—the problem, the user, the main use cases, the scope, and what I wanted to achieve with the first version.

From there, I created user stories. That helped me look at the product from the user's perspective rather than thinking about how I was going to implement it. I then organised the stories into an initial backlog and broke the work down into tasks that I could eventually implement.

```mermaid
flowchart LR
  Idea[Vague idea] --> Brief[Product brief]
  Brief --> PRD[PRD V1]
  PRD --> Stories[User stories]
  Stories --> Backlog[Backlog and release cuts]
  Backlog --> Tasks[Implementable tasks]
```

There was another reason I wanted to document all of this: **I knew I would be using AI tools to help me build the project.** Instead of explaining the project from scratch every time I asked an AI coding tool for help, I could give it the product documents as context. The PRD and user stories became a shared reference point—for me as the developer, and for the AI tools I was working with.

```
docs/
├── README.md
└── product/
    ├── product-brief.md
    ├── prd-v1-memory-capture-recall-summary.md
    ├── user-stories-v1.md
    ├── backlog-and-release-cuts.md
    ├── metrics-and-evaluation.md
    └── risks-assumptions-open-questions.md
```

The idea started in my head as something fairly vague. By the end of this process, I had something much more concrete: **an idea → a product definition → user stories → work I could build.**
