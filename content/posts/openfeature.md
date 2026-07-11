---
title: "The cloud foreigner's guide to OpenFeature"
draft: true
---

Look, it's actually quite simple. Having either erected an evaluation engine
external to your process, or embedded it within your process (synched via
either GRPC or HTTPS to one of the myriad compatible persistence layers), you
have merely to import the appropriate provider for your language, configure the
OpenFeature SDK to that provider via a mutation function, and run
instance-methods via the SDK to request values for named flags, providing, if
appropriate, the evaluation context relevant to that flag's targeting.

You're welcome. Easiest blog post ever.

What's that? You require further explanation? You must not be cloud native.
What are you then? Some kind of cloud foreigner? That's cool, no worries. I
wasn't born here either. Let me back up for a second and begin somewhere closer
to, well, the beginning.

## Feature flagging? You mean like `--help`?
