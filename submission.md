# Submission
---
The project that I worked earlier that I'm very proud of is 'Eric.ai', with the company 'Combine ai'

## Backend Migration: Python -> Ruby, Claude -> openAI 

**For a non-technical audience**
I worked on Eric AI as part of a small engineering team building an AI-powered system that connects Microsoft services and large language models (including OpenAI’s GPT) to deliver a simple and reliable user experience.

A major responsibility I owned was leading a full migration of a core backend system from Python to Ruby. This was a high-impact change that required careful coordination to ensure the product continued running smoothly without disruption. I worked closely with multiple stakeholders across design, product, and analytics teams to ensure the migration aligned with user needs, business priorities, and expected outcomes.

As part of this work, we also migrated from an older Anthropic Claude-based setup to OpenAI’s GPT models, which improved performance and reduced operational costs by approximately 15%.

This project required strong cross-functional communication, end-to-end ownership from planning through execution, and delivering changes safely in a production system. I was recognized for efficiency and successful delivery of the migration work.

---
**For a technical peer**
I led a full migration of a core backend service from Python to Ruby, aligning it with the rest of the backend stack and improving long-term maintainability. This involved building out the new service while making sure it stayed compatible with existing systems and external integrations.

One of the harder parts was making sure nothing broke during the transition. I had to carefully validate that both versions behaved the same, roll things out in stages, and keep production stable throughout the process while coordinating changes across dependent services.

Beyond just matching functionality, I also had to think about how the system would behave in production over time — things like scalability, reliability, and observability. I paid attention to performance (latency and resource usage) and made sure failures were handled cleanly implementing retry logics along with background jobs.

Because parts of the system handled concurrent requests, I also had to think about thread safety and data consistency, and used locking where needed to avoid race conditions.

On the AI side, I worked directly with OpenAI’s APIs and focused on prompt engineering for production use. I tuned things like temperature, top_p, max_tokens, and repetition penalties to get more consistent and useful outputs depending on the use case. Model choice was based on tradeoffs between cost, latency, and output quality.

---
## Summary

Overall, the work was a mix of backend migration, production reliability work, and AI integration — with a strong focus on keeping the system stable, efficient, and maintainable in real-world use.
