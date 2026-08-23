# Plans, Usage & Limits

Claude plans decide what level of access we have. Usage limits decide how much we can use that access over time. Length limits decide how much information Claude can hold in one conversation.

These are easy to mix up, so it helps to separate them.

## Plans: Which Level of Access Do We Need?

Claude offers plans for occasional use, regular individual use, heavier individual use, and teams or organisations. The exact features, prices, and availability can change by country and over time, so we should treat the Claude upgrade page as the source of truth.

In simple terms:

| If our work looks like this                                    | We should consider             |
| -------------------------------------------------------------- | ------------------------------ |
| Occasional questions, learning, and light experimentation      | A free or entry-level plan     |
| Regular research, writing, and coding                          | An individual paid plan        |
| Daily, long-running work across Claude, Claude Code, or Cowork | A higher-usage individual plan |
| Shared controls, billing, and organisational needs             | A team or enterprise plan      |

We should choose based on the work we actually do, not only on the headline message count.

## Usage Limits: Our Budget Over Time

A usage limit controls how much we can interact with Claude during a period before waiting for a reset or using available credits.

Usage is affected by more than the number of messages we send. A short conversation about a simple task uses far less than a long conversation with large files, tools, and deep reasoning.

The main factors are:

* the length of our messages and attachments;
* the amount of conversation already in the session;
* the model we choose;
* the effort or thinking level we use;
* tools such as web search, research, or connectors;
* artifact creation and other product features.

{% hint style="info" %}
On many paid plans, usage across Claude.ai, Claude Code, and Claude Desktop contributes to the same allowance. We should keep that in mind when a heavy coding session seems to affect chat availability.
{% endhint %}

## Length Limits: Claude’s Working Memory

A length limit is different. It relates to the model’s context window: the amount of information Claude can work with in a single conversation.

That context may include our messages, uploaded files, project instructions, tool results, and Claude’s previous replies. Part of the window is reserved for the next response, so the usable space is always a little smaller than the headline context-window number.

When we reach a length limit, starting a fresh conversation or splitting the work into smaller tasks is often the right move. This is a context-management problem, not necessarily a plan problem.

## How We Use Our Allowance Well

We do not need to become obsessed with tokens. A few simple habits are usually enough:

1. Start a fresh conversation when we switch to an unrelated task.
2. Put stable background information in a Project instead of uploading or repeating it every time.
3. Keep project instructions short and useful.
4. Attach only the files that matter to the current question.
5. Turn off tools or connectors that we do not need.
6. Use higher effort and premium models for work that genuinely needs them.
7. Check **Settings → Usage** before assuming something is broken.

These habits improve both usage and answer quality.

## When We Hit a Limit

| What we see                                       | What it usually means                              | What we can do                                                    |
| ------------------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------- |
| A usage-limit warning or reset time               | We have used the allowance for the current period  | Wait for the reset, change plans, or use credits if available     |
| A conversation cannot accept more information     | The current context is full                        | Start a new conversation, simplify the context, or split the work |
| Answers become less focused in a very long thread | Important context may be buried under old context  | Summarise the state, then continue in a fresh conversation        |
| A costly workflow uses too much allowance         | The model, tools, or task shape may be inefficient | Review the workflow before simply upgrading                       |

## Plans Are Not the Same as API Billing

Claude subscriptions and the Claude API are separate products. A subscription is for using Claude products such as Claude.ai and, depending on the plan, Claude Code or Cowork. API usage is billed separately through the Anthropic platform.

When we build an application with the API, we should use the API pricing and rate-limit documentation—not the subscription-plan limits on this page.

## Official Documentation

* [Claude Help: Choose a Claude plan](https://support.claude.com/en/articles/11049762-choose-a-claude-plan)
* [Claude Help: Usage and length limits](https://support.claude.com/en/articles/11647753-how-do-usage-and-length-limits-work)
* [Claude Help: Usage-limit best practices](https://support.claude.com/en/articles/9797557-usage-limit-best-practices)
