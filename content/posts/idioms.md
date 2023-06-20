---
title: "Go-ing Idiomaniacle"
date: 2022-10-23T12:35:20-06:00
tags: ["Golang", "pontificating", "programming", "philospohy"]
draft: true
---

# Idioms

In all human endeavor there is an all-encompasing list of things you can do, which is composed of the subset of things you _should_ do, and the subset of things you _should not_. In software engineering we call the former subset -- the one containing things you should do, _idioms_. 

Correct code is said to be "Idiomatic", by which we mean, the code is a marriage of valid syntax and correctly applied idioms. These are orthagonal properties. Code may posess perfectly valid syntax and perform the task it was written to accomplish, and still be _incorrect_, if the idiomatic layer has not been properly applied. As engineers, we typically express this state with olifactory metaphors eg.. "code smell".  If, therefore, you endeavor to be a "good" engineer -- one who writes correct, and odorless code -- you will need to concern yourself with the comprehension and proper application of idioms.

Compared with idioms, programming languages are easy to learn. Simply pick up the book or spec, and commit the syntax to memory.  Idioms by comparison are rarely documented, language specific, often counterintuitive and sometimes political and/or controverseal. They don't generalize because ultimately they express someone's opinion. By using the word [idiom](https://www.merriam-webster.com/dictionary/idiom) to describe them, we're outwardly acknowledging these properties. They can't be understood by themselves or as a function of their syntax. They are subjective, and derive from human-oriented contexts.

Worse still, as the language grows and opinions change, the idioms will evolve. Unlike syntax, which will remain valid until it is very explicitly deprecated, it's entirely possible to discover and ingest "old", invalid idioms as your reward for otherwise studious research. In fact many of the skills that might make you good at learning a language, can work against you in the context of internalizing its idioms.

Let's look at some examples, I'll pick on Golang since its my day-to-day, but I think you'll find examples of these in every language more or less.

## Readability

Many exist for the subjective purpose of "readability". That *operators (+-=) should have a single space on either side*, for example, is subjective but easily enforced with the use of a linter. Lately languages have even been shipping with `fmt` tools, to enforce the communities cherished beliefs around "readability".  Level 1 idiomatical correctness is to ensure you're using the linters and fmt tools available for the langage you're writing in.

## Performance

Others exist for more complex reasons, can't really be linter-enforced, and even when documented can be more difficult to apply. Golang's *exit early* idiom is a good example of this. About 99% of the time, an `else` clause in a conditional statement could have been avoided with an early return. Early-return is a good performance habit, which the Go writers thought was important enough to call out in their seminal doc on idioms: [Effective Go](https://go.dev/doc/effective_go)

## Uniformity

Many assume that "Professional" programming must me more complex and clever than casual code-writing, but this the opposite of reality. Rather, writing industrial-strength code is mostly an excercise in removing the unexpected. To the extent that someone else can open your source-code and quickly intuit the patterns of thought behind the functions and classes you've written, your code is "professional". To this end, idioms serve to ensure that you're thinking within the lines of the same "patterns" everyone else is, and if that sounds to you like idioms enforce homogeniety of thought, you're absolutely right, that's maybe their defining feature.

An interesting nuance here is that the element of "suprise" invariably changes whenever you change teams (even within the same company). I think one of the things that probably defines you as a staff or principle level engineer, is that you objectively understand whether your code *should be* suprising to your new team. Because in the end, if you join a new team, and your code is suprising to them, it's either because you need to become more idiomatic, or they do.

But I digress. By way of an example, there are many creative ways to manage goroutine lifecycle in golang for example, but idiomatically, you'll want to have a bullet-proof reason to be using anything other than waitgroups or errorgroups.


## Plaster

Some idioms make up for, or plaster-over deficienies in the underlying language. The go idiom of immediatly following an http.Client constructor with defer .Close(), or the idiom against using time.After() like _ever_, both plaster-over what could be argued are memory-leak-by-design problems in the http and time packages.

## Hazing

Idiom's are the spanking paddle of the code review hazing ritual, and to a certain extent I guess that's fine. The general notion behind hazing is to create a bond, born of commonality of suffering. I want to know that you've suffered as I have, and together we'll overcome, or at least.. ya know.. persist.

The point where it becomes objectionable is when I begin to enjoy your suffering for the sake of it, which, I think is an invariant side-effect. Athelets enjoy making other atheletes suffer because it makes them feel stronger-than, and engineers enjoy it because it makes them feel smarter-than. And hey, it feels great to be smarter-than, I know, but that little dopamenergic spike of elation you're feeling when you ding someone on non-idiomatic code is illusory. It's too cheap in the first place -- too easy to spot -- literally anytime you see `else` in a Go program etc... That act requires no cognition or effort on your part, which you should know by now, draws its value severely into question. When was the last time doing an effortless thing made you smarter-than?

The danger here is not only that you'll alienate another engineer with a laundry-list of comments that amount to nitpicky bullshit, but also that you'll attune to it -- that your code reviews will become an excercise in hunting for easy to spot, easter-egg-like idiomatic mistakes. In short, that you will become: idiomaniacle. To be clear I'm not saying you shouldn't be commenting on them, but I do believe those comments should come from a place of commiseration rather than superiority if you expect them to do any good.

take it easy
-dave
