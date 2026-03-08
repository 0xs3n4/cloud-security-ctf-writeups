# Infinity Learning Defensive CTF — Investigating S3 Object Brute Forcing with Elastic

## Overview

This write-up covers my experience solving a **defensive cloud security CTF** from **Infinity Learning**, where the goal was to investigate suspicious activity involving **object brute forcing against an S3 bucket** using **Elastic / ELK**.

This challenge was less about exploitation and more about **log analysis, detection mindset, field discovery, and learning how to navigate cloud telemetry under pressure**.

It was also a good reminder of something important:

> In defensive CTFs, sometimes the hardest part is not understanding the attack itself — it is understanding how the logs are structured and where the useful evidence actually lives.

---

## Challenge Context

The CTF provided access to **Elastic** and the scenario was focused on identifying brute force behavior against objects inside an AWS S3 bucket.

At the start, the challenge asked two direct questions:

1. **Identify and determine the `user_agent.original` associated with the Object Brute Forcing activity.**
2. **Identify the bucket that is targeted for object brute forcing activity.**

At first glance, these looked simple. In practice, this turned into an exercise in **log exploration, field mapping, and persistence**.

---

## Initial Difficulty

I had already completed other defensive CTFs using **Elastic / ELK**, but I still do not consider myself fully fluent in the platform. I can move around, search, and investigate, but I am still building that “analyst instinct” inside the tool.

So this challenge started with a familiar feeling:

- I understood the detection goal.
- I understood the AWS context.
- But I did **not** immediately know the fastest path to the answer inside Elastic.

That gap mattered.

Instead of jumping straight to the correct field, I spent time exploring different events and trying to reason through the attack from the telemetry outward.

---

## My Initial Approach

Since the challenge was about **object brute forcing in S3**, my first thought was to investigate event patterns that could indicate repeated object discovery attempts, such as:

- bucket-level probing,
- object existence checks,
- repeated access attempts,
- unusual request sources,
- scripted tooling.

Because I was still thinking in terms of **attack behavior first**, I started looking through events that seemed related to S3 reconnaissance.

One of the first places I looked was around **`HeadBucket`-style activity**.

That made sense conceptually, but it did **not** immediately give me the answers I needed.

I also tried filtering by **user agent**, since one of the questions explicitly asked for `user_agent.original`. That was a useful instinct, but I still did not have enough precision to isolate the exact activity cleanly.

---

## Asking for a “Defensive HackTricks”

At that point, I realized I needed to slow down and re-center.

So I asked ChatGPT for something like a **mini defensive playbook**, almost a “HackTricks for blue team investigation,” focused on:

- how to think about brute force against S3 objects,
- what fields are usually most useful,
- how repeated failed requests may appear,
- what to inspect in logs when the answer is not obvious.

That helped me mentally structure the hunt.

It did not magically solve the challenge, but it gave me a better workflow:

1. Identify the likely service.
2. Understand the event shape.
3. Inspect available fields instead of guessing.
4. Pivot from one event to related evidence.
5. Treat the sidebar fields in Elastic as part of the investigation, not just UI decoration.

That last point turned out to be the key.

---

## The Wrong-but-Useful Detours

One thing I liked about this challenge is that even my wrong turns were still educational.

### 1. Looking at the wrong event types
I explored events that looked relevant from an AWS perspective, but they were not necessarily the **best evidence source** for the questions being asked.

This is common in cloud investigations:
you know the service, but not every log record is equally useful.

### 2. Finding the user agent... but not exactly the way the challenge expected
At one point I found the user agent value:

`censored`

That was already a very strong indicator of brute forcing or wordlist-driven probing.

But the challenge expected it in a slightly different representation, where it appeared wrapped with square brackets:

`censored`

That small detail was funny and slightly annoying at the same time.

It was one of those classic CTF moments where you technically have the right answer, but formatting still matters.

### 3. Over-focusing on assumptions instead of fields
Another lesson was that I initially spent too much time trying to infer where the answer *should* be instead of using Elastic to confirm what fields were actually present in the event.

That was the turning point.

---

## The Breakthrough

While navigating the event details in Elastic, I noticed something extremely useful:

Elastic shows a **sidebar with all populated fields for the selected event**.

That changed the game.

Instead of continuing to search blindly, I started using the sidebar to inspect the actual structure of the event and determine:

- which fields existed,
- which ones were populated,
- which ones were empty,
- and which field contained the richest raw evidence.

That process led me to the field:

**`event.original`**

And that was the key.

---

## Why `event.original` Mattered

The `event.original` field returned a **raw JSON blob**, very similar in spirit to what you often see in AWS CloudTrail-style data.

Inside that raw event payload, I was finally able to extract the answers clearly.

That field contained the details I needed, including:

- the **user agent** associated with the brute forcing activity,
- and the **targeted bucket name**.

This was the moment the challenge clicked.

Not because I had invented a clever detection query, but because I finally stopped fighting the data and started reading the event in its most complete form.

---
```text
