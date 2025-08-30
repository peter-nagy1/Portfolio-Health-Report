# AI Choice

For the Portfolio Health Report proof-of-concept, we used the following AI models:

### 1. OpenAI `gpt-4o-mini`
I would use this as the primary model for the task of attention flag detection in email threads due to it's performance, ability to scale, and price per tokens.

**Justification:** It has high-quality reasoning capabilities which is needed to understand multiple back and forth emails, good summarization capabilities which would be helpful when we have a long tread of emails and we need to summarize part of it, and reliable JSON output formatting.

We can scale our model up by using `gpt-4o` which comes at a higher cost for better performance.

### 2. Gemini `gemini-2.5-flash`
I added this as an alternative model to test the system without relying on OpenAI credentials.

**Justification:** It supports free-tier usage for testing and thus allows the proof of concept to be functional without the need to use any mockups. It also provides good back and forth conversation reasoning and reliable JSON output formatting.

I've also used `gemini-2.0-flash-lite`, `gemini-1.5-flash` to test the system.