---
title: the skill I made so AI writes the way I do
description: the first version of humanizar still let AI-sounding text through. what I kept pointing out on my portfolio bio, round after round, became the new version
date: 2026-10-09
---

# the skill I made so AI writes the way I do

## tl;dr

**humanizar** is a skill for AI agents (Claude Code, Codex, Cursor) that rewrites text that reads like AI and writes new text in the voice of whoever signs it. the first version had the usual list of tics ("leverage", "journey", em dashes everywhere) and still let a lot through. I used it to fix the copy on my portfolio and kept pointing out what was wrong, round after round. each point became a rule. the new version is in my skills repository, in English and Portuguese.

## the problem

removing "leverage" from a text is easy. the hard part is what's left: a clean text, without a single banned word, that still sounds like AI. it's vague and it talks in a way I never would.

the bio on my portfolio's home page was the test. it took about six rounds. the bio is in Portuguese, so I translated the quotes below.

## round by round

**1. the starting point.** the bio said:

> I work at Loopscape building products end to end. Front-end is my thing, but I go from the database and API all the way to the last screen state.

and, a bit further down, "nothing goes into a project until I understand what the code is doing". there isn't a single inflated word in there, and I still found it raw, AI-sounding. the problem was something else: "from the database to the screen" sounds broad and shows nothing, and the last sentence is a motto, the kind that fits on a mug.

**2. the reference.** I sent someone else's bio, in English, and said that was the idea. the AI copied the paragraph structure, translated an expression almost word for word ("software people spend their whole workday in") and nearly brought along facts that aren't mine, like ERPs and an X account. it wasn't supposed to copy. it was supposed to take the spirit.

**3. the clever image.** the next version had "I like the part that never shows up in a screenshot: the loading state, the error message, the screen that opens fast". I didn't understand what "the part that never shows up in a screenshot" meant. if I didn't get it, readers wouldn't either. it had two more problems: a colon followed by a list of three, and the wrong grammatical gender for Loopvet in Portuguese.

**4. the jargon.** then came "loading and error states", "a new API route", "a database change". all true, but that's not how I introduce myself. what I'm into is motion and design; the rest is the rest.

**5. the word.** "the lab on this site is where I play with that" became "where I show that". a small detail, but the word has to be the one I'd use.

**6. the line about AI.** to say how I use AI, the skill gave me three options and I didn't want any of them. I wrote it myself, and that's the one that stayed.

the bio that stayed:

> I'm a junior front-end dev at Loopscape, working on Loopvet, a system for veterinary clinics. What I enjoy most is motion and design, and the lab on this site is where I show that.
>
> In my free time I work on cd/ui and converter-hub and study security with Sentinel Forge. I use AI every day to learn faster, but I try hard to understand what's going on in the code and I review a lot. I'm still learning what works best.

## what changed in the skill

each round became a rule:

- **find the voice before writing.** the skill reads 2 to 4 texts the person has already published and notes concrete traits, like which contractions they use and which words they'd never say. chat is a step looser than the site: I type slang in chat that never goes on my portfolio.
- **inspire mode.** when you send someone else's text, it says in one line what the reference does well and writes that with your facts. there's a test at the end: reading both side by side, can you tell one was built on top of the other? if so, rewrite.
- **facts need a source.** a concrete detail only goes in if it came from the original text, the project files or what the person said. with no source, it asks. that's what was missing with the Loopvet mistake.
- **words the person would say.** in bios and website copy, no jargon they wouldn't use in conversation. technical detail goes to the case study.
- **short text gets 2 or 3 versions**, each with an angle, so I pick instead of going back and forth.
- **a final check**, run on its own text before delivering: a punchline at the end of a paragraph, a colon with three items, "from X to Y", a fact without a source.

the tic catalog got a new section, the tics that survive a first revision. almost every example in it came from this bio.

## what I learned

- the "before" didn't have a single banned word. the problem was being vague, and a list of banned words doesn't catch that.
- AI gets the generic developer tone right, not mine. it only gets close when it reads what I've already written.
- the line about AI in my bio is one I wrote, after turning down three options. the skill helped clean up the rest and helped me see why its versions didn't sound like me.

the skill lives in my skills repository: [github.com/di0rio/cd-skills](https://github.com/di0rio/cd-skills/tree/main/skills/humanizar), and on the [/skills](https://cauadiorio.vercel.app/en/skills) page of my portfolio.
