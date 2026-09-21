# AI Case Studies

Most of this site is about infrastructure. This section is about **working with an AI on infrastructure**, which turns out to have its own failure modes, its own tells, and its own techniques.

These are not "look what the AI did" posts. They are the opposite: cases where the AI was confident and wrong, where the human's scepticism was the thing that unstuck the problem, and where it is possible to point at the exact sentence that changed the outcome.

The reason to write them down is that the useful skill here is not prompting. It is **knowing when to refuse an answer**, and having a few reliable ways to make the model go and look again.

## Index

- [Pushing back: the four words that ended a 15 hour outage](2026-09-20-pushing-back-on-the-ai.md). An AI concluded a fix required physical access, correctly, from facts that were all true. The conclusion was still wrong. Four words broke it open, 13 minutes later the box was fixed, and a second AI caught that the first answer would have bricked it.

## The short version, if you read nothing else

Patterns that repeatedly produced better answers than accepting the first one:

| Move | What it sounds like | Why it works |
|---|---|---|
| **Refuse the dead end** | "really no way of hacking it eh" | Models state conclusions with the same confidence as facts. Rejecting the conclusion while accepting the facts forces a re-derivation. |
| **Contradict with lived knowledge** | "I've never really changed anything in the bios" | You hold evidence the model cannot observe. It will build confident theories that your own history rules out. |
| **Correct its account of itself** | "yez but you ran it yesterday too check it pks" | A model's memory of its own prior actions is unreliable. It will deny doing things it did. |
| **Demand adversarial review** | "have a codex agent check it out too" | A second model with no stake in the first one's plan catches what the author cannot. This one prevented a bricked machine. |
| **Ask who else hit this** | "has anyone else complained about this on the internet" | Moves it off first-principles reasoning and onto evidence other people have already gathered. |
| **Ask it to investigate itself** | "i feel like it was something you did at some point" | Models do not volunteer their own culpability. They will report it accurately when asked directly. |

None of these require knowing more than the model. They require being unwilling to stop.
