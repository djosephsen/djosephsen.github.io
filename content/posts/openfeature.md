---
title: "The cloud foreigner's guide to OpenFeature"
date: 2024-11-10:36:53-07:00
draft: true
---

Look, it's actually quite simple. Having either erected a flag management
system external to your process, or embedded an evaluation engine within your
process (synched via either GRPC or HTTPS to compatible persistence layer), you have
merely to import the appropriate provider for your language, configure the
OpenFeature SDK to that provider, and run instance-methods via the SDK to
request values for named flags, providing, if appropriate, the evaluation
context relevant to that flag's targeting.

You're welcome. Easiest blog post ever.

## Did you just tell me to go screw myself?

What's that? You require further explanation? You must not be cloud native.
What are you then? Some kind of cloud foreigner? That's cool, no worries. I
wasn't born here either.
