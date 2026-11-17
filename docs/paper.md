# Cordon: keeping AI agents from leaking secrets, without asking another AI to stand guard

*A plain English write-up. November 2026.*

## Summary

AI agents can be tricked by hidden instructions in the things they read. When that happens, the worst outcome is usually the same: the agent reads something private and sends it to the attacker.

Most defenses try to detect the trick. Cordon takes a different path. It sits between the agent and its tools and enforces one rule: an agent that has read both untrusted content and private data may not use any tool that sends data out. The rule is plain logic, with no AI involved, so it can't be talked around.

In my tests, a scripted agent that fully obeys a hidden instruction leaked a secrets file in watch-only mode and leaked nothing with Cordon switched on. A second part of Cordon, which recognizes private data inside outgoing messages, found every one of 480 randomly generated leaks across 8 kinds of scrambling, with no false alarms on 200 unrelated messages. Every decision took under 2.5 milliseconds on a normal laptop.

This write-up explains the problem, the idea, how Cordon is built, how I tested it, and where it falls short.

## 1. The problem

An AI model reads text and decides what to do next. The trouble is that everything looks like text to it. Your instructions are text. A web page is text. An email from a stranger is text. The model has no reliable way to know which text is the boss.

So an attacker doesn't need to hack anything. They just need to put some words where the AI will read them. A comment on a web page. A line in an email. A note inside a bug report. This is called prompt injection.

On its own, a tricked chatbot is mostly embarrassing. A tricked **agent** is a different story, because agents have tools. They can open files, query databases, send emails and make web requests. In 2025, researchers at Invariant Labs showed that a single malicious issue on a public GitHub project could steer an AI agent into pulling data out of the user's private projects [4]. Nothing was broken into. The agent was simply asked nicely by the wrong person.

## 2. Why "just detect it" isn't enough

The obvious fix is to scan for bad instructions, often with a second AI. This helps, and it's worth doing, but it has a built in weakness.

A detector that catches 99 out of 100 attacks sounds great until you remember that attackers get unlimited tries. They can test a thousand wordings in an afternoon. A defense that works "most of the time" is a speed bump, not a wall.

There's also a deeper issue. If the guard is an AI, the guard reads text too, and can be fooled by the same kind of trick.

## 3. The idea: take away one leg of the stool

The security researcher Simon Willison gave the dangerous setup a name: the **lethal trifecta** [1]. An agent is in trouble when it has all three of these at once:

1. It reads content that strangers can write.
2. It can get at private data.
3. It can send data to the outside world.

With all three, an attacker can plant an instruction (1) that makes the agent fetch a secret (2) and ship it out (3). Take away any one and the attack falls apart, like a stool with a missing leg.

Meta published the same idea as a design rule called the **Agents Rule of Two**: an agent should have at most two of these three powers in one session [2]. A research team has also shown a more advanced design, called CaMeL, that carefully tracks where data is allowed to flow [3].

These are great ideas. What I wanted was something small and practical that you could drop in front of an existing agent today, without rebuilding the agent. That's Cordon.

## 4. What I assume about the attacker

It helps to be clear about what Cordon is and isn't defending against.

**The attacker can:**

- Put any text they like into things the agent reads: web pages, issues, emails, documents.
- Completely take over the agent's behavior with that text. I assume the trick always works.
- Make the agent scramble data, split it up, or try several tools.

**The attacker cannot:**

- Change Cordon's settings or the machine it runs on.
- Reach tools that aren't connected through Cordon.

**What I'm protecting:** private data must not reach the outside world because of something a stranger wrote.

Notice the second point in the first list. I don't assume the AI is smart enough to resist. I assume it loses every time. If Cordon holds up under that assumption, it holds up for every smarter model too.

## 5. How Cordon works

There's a longer tour in the [architecture page](architecture.md). Here's the short version.

**It's a middleman.** Agents reach their tools through a standard called MCP [5]. Cordon speaks MCP on both sides, so the agent talks to Cordon and Cordon talks to the tools. Every tool call has to pass through it.

**You label your tools.** In a short settings file, each tool is marked as "reads from strangers", "reads private stuff", "sends things out", or some mix.

**It keeps two switches per run.** One flips when the agent reads something from a stranger. The other flips when it reads something private. They never flip back during that run.

**It applies the rule.** When both switches are on, every "sends things out" tool is closed. The agent gets a clear "blocked, and here is why" message. Everything else keeps working.

**It collects evidence.** Cordon remembers fingerprints of the text that labelled tools have returned. When the agent tries to send something out, Cordon checks if chunks of that text are inside, including when they've been scrambled as base64, hex or web encoding, even in layers. This lets it say "this message carries 243 bytes from your secrets file" instead of just "blocked".

**It writes everything down.** Each decision goes into a log where every line is chained to the one before. Edit, delete or reorder a line and the chain breaks at that spot. Secrets themselves are never written into the log.

One design point matters more than the rest: **the evidence is not the lock.** Recognizing scrambled data is a cat and mouse game, and I don't want safety to depend on winning it. The rule in step four doesn't look at the message at all. Even if the attacker invents a scramble Cordon has never seen, the call is still blocked. You just get a less detailed explanation.

## 6. How I tested it

### 6.1 A full attack, start to finish

I built a small test setup: three tool servers (a bug tracker, a file reader, and a web tool), plus a scripted agent that does exactly what a hidden instruction tells it. The messages between them are real MCP messages going through the real Cordon gateway. Only the "brain" is scripted, and it's scripted to be fully fooled.

The attack goes like this. A bug report contains a hidden note telling AI assistants to read the secrets file and send it to an address. The agent reads the report, reads the secrets file, then tries to send the contents out as base64. When that's done it tries again with a different tool, sending one key as hex inside a web address.

| | Watch-only | Switched on |
| --- | --- | --- |
| Attempts to send secrets out | 2 | 2 |
| Attempts that got through | 2 | 0 |
| What the attacker received | The whole secrets file, then one key again | Nothing |
| Could the agent keep doing normal work? | Yes | Yes |

In both modes Cordon correctly identified that the outgoing messages contained data from the secrets file, named the scrambling used (base64, then hex), and traced the destination address back to the bug report that planted it.

### 6.2 Can the evidence finder be relied on?

I wanted more than a couple of hand picked examples, so I wrote a randomized test. Each round it:

1. Makes up a random "secret" between 200 and 1,700 characters long.
2. Picks a random chunk of it, between 19 and 118 characters.
3. Scrambles the chunk one of 8 ways: left as is, base64, the web safe kind of base64, hex, web address encoding, base64 inside web address encoding, base64 done twice, or base64 tucked inside a structured message.
4. Buries the result in random noise.
5. Checks that Cordon finds it.

Result: **480 out of 480 found.** In a separate test, 200 random messages with no connection to the secret produced **zero false alarms**.

I also checked that when the data is private, Cordon reports how much it found but never copies the private text into its own records.

### 6.3 Speed

In the attack runs, every decision took under 2.5 milliseconds on an ordinary laptop, and most took far less than one. For comparison, a single AI model call usually takes hundreds of milliseconds to several seconds. So Cordon adds nothing you would notice, and since it makes no AI calls it adds nothing to your bill.

### 6.4 The log

I tested the chained log by editing a line, deleting a line, and swapping lines around. Each time, the check failed and pointed to the right spot.

### 6.5 Old and new MCP

MCP changed a lot in its July 2026 version [5]. I tested Cordon with a client and servers speaking the older style, the newer style, and a mix of both in the same run. All combinations worked.

### 6.6 What these tests don't show

All of this used my own test agent and my own test tools. That's good for proving the logic, but it isn't the same as months of use with popular agent products and real tool servers. I haven't done that yet, and I don't want to pretend otherwise.

## 7. Limits

**It only guards its own door.** Tools that the agent reaches some other way, like a built in web browser or a built in command line, never pass through Cordon. For Cordon to mean anything, the risky tools have to be connected through it.

**The labels are on you.** Cordon trusts your settings file. A tool in the wrong bucket, or a tool you forgot to label, is a hole. The dashboard lists unlabelled tools so they don't go unnoticed, but it can't know what a tool really does.

**It's blunt.** After the agent has read both stranger content and private data, it can't send anything out for the rest of the run, even for an innocent reason. That's the price of a rule that can't be argued with. A future version could let a human approve a specific call.

**A run is one launch of the gateway.** The newest MCP has no built in idea of a "conversation", so Cordon can't tell when a new chat starts. If one gateway stays open across many chats, the switches stay on. This makes Cordon stricter than it needs to be, never looser.

**The evidence finder can be beaten.** An attacker can invent scrambles it doesn't know. The block still happens, because the rule doesn't depend on the finder, but the explanation will be thinner.

**The log shows tampering, it doesn't prevent it.** Someone who can rewrite the entire file could build a fresh, consistent fake. The usual fix is to copy the newest fingerprint to a safe place from time to time.

**It doesn't stop every kind of harm.** Cordon is about data leaving. It doesn't stop a tricked agent from, say, deleting files or giving you bad advice.

## 8. What's next

- An "ask a human" option, so a blocked call can be approved once instead of flatly refused.
- Testing with the popular agent products and common tool servers.
- A way to reset the switches per conversation, for apps that can tell Cordon when a new one starts.
- Support for tools that are reached over the network, not only ones running on the same machine.
- Saving log fingerprints somewhere safe automatically.

## References

1. Simon Willison. "The lethal trifecta for AI agents: private data, untrusted content, and external communication." June 2025. https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/
2. Meta. "Agents Rule of Two: A Practical Approach to AI Agent Security." 2025. https://ai.meta.com/blog/practical-ai-agent-security/
3. Edoardo Debenedetti and others. "Defeating Prompt Injections by Design." 2025. https://arxiv.org/abs/2503.18813
4. Invariant Labs. "GitHub MCP Exploited: Accessing private repositories via MCP." 2025. https://invariantlabs.ai/blog/mcp-github-vulnerability
5. Model Context Protocol specification, version 2026-07-28. https://modelcontextprotocol.io/specification/2026-07-28

---

Copyright (c) 2026 marpusakethram. All rights reserved.
