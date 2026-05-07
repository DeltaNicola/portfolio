---
layout: single
title: overcomplicating RAG cause I feel like it
excerpt: how I ended up with three collection types when one would've done the job
date:   2026-05-07 15:00:00 +0200
tags: [article, other]
toc: true
toc_label: "indice"
toc_icon: "heart"
toc_sticky: true
---

I started building `ragd` as a normal RAG system

vector store, embeddings, chunker and retrieval, basically the standard stack  
the original use case was a security knowledge base for [deltarecon]({% link _projects/deltarecon.md %}), and a single read-only collection of markdown documents would have been way more than enough

but I like overcomplicating things, and I also like to see how far I can push some ideas

## introducing ragd

`ragd` has three collection types now, two of which are not strictly required by the original use case, and one of which I'm still discovering how to use properly  
this post is mostly about how I got there and what those collection types achieve

### knowledge

the security KB part is genuinely simple  
markdown files in a repo, `ragd` indexes them, the pentesting pipeline queries them at runtime to retrieve relevant attack-taxonomy passages for the LLM analysis pass

this is the use case `ragd` was built for and it's the part that runs at scale: three `ragd` instances behind a load balancer, real query volume from concurrent pipeline runs, reindex when the source changes  

if I had stopped there, `ragd` would be a small focused tool with one collection type, which I called `knowledge`, that supports ingestion, indexing, and retrieval, and that's it

the agent never writes back to it, the source is the markdown in the repo and the index is just a derived view that helps with retrieval

### pages

at some point while building `ragd` I started thinking about a different problem

the security KB is fine: it's reference material, the agent consults it, it doesn't change much 

but I was also thinking about agents that needed to remember things across sessions
notes about projects, decisions made, structured state that evolves

the kind of thing a person would write down in a notebook and update over time

a vector store can technically hold this, but the operations don't fit: if the agent has a "page" about project X and wants to update it, what's the operation?

delete the old chunks, insert new chunks? what if two parallel sessions try to update at the same time? what if the chunks for that page were embedded with a model you've since changed?

these are real problems and the vector-store-as-everything design doesn't have clean answers for them

so I added a second collection type, `pages`, with different rules

pages are addressable by id and they're stored in sqlite as the canonical record, with a derived vector index for retrieval

updates use an ETag protocol: read returns content + ETag, write requires the current ETag, mismatch returns a 409 conflict and the agent has to re-read and try again

optimistic concurrency, basically, the same pattern HTTP has had for forever

was this necessary for the security KB use case? no  
was it necessary for any use case I was actively running? also no, at the time  
I built it because I wanted the option, and because it was 2AM and I needed something to do

### journal

a while later I wanted ordered event logs

pages handle "what does the agent currently believe about project X" well enough but they don't handle "what did the agent do this session, in order" because pages (in my design, at least) are mutable and addressable, not append-only and ordered

I could have used pages for this: write a new page per event, give it a timestamped id, query in order

thing is, it seemed like the fastest way to end up with tons of small pages that would have oversaturated the sqlite very fast

also, pages allow update and delete, which is wrong for an event log: yesterday happened the way it happened, and a system where the agent can rewrite the history of what it did is not really a log, it's just whatever was written last

so a third type, `journal`, has spawned into my mind

append-only, ULID-keyed (lex-sortable, millisecond-precision, collision-resistant across processes), with a `since` cursor for replay  
no update operation, no delete operation, type system enforces append-only

was this necessary? again, no, not for any use case I was running

I'm still discovering what kinds of agentic workflows benefit from a real journal vs. just keeping conversation history in memory but the option exists and I'm okay with that

## an honest discussion

when I started working on this, the security KB use case alone would not have justified building `pages` and `journal`  
truth be told? the goal of `ragd` evolved as I kept working on it

at some point, I wanted `ragd` to become the memory layer for agentic workflows I'd build later, not just the retrieval layer for the pentesting pipeline I had in front of me

which means building the shapes I'd want to use eventually, even if I wasn't actively using them yet
the agentic workflows are coming and when they will need memory, `ragd` will have the shapes ready

I also learned things by building this that I wouldn't have learned by building just the simple version: metadata flattening problems with chroma, the round-trip issues with structured fields, the fact that vector stores are great as indices and weak as systems of record, the way ETag-style optimistic concurrency falls out naturally when you treat sqlite as canonical

none of this is new: these are well-trodden patterns in database design, but applying them to "memory for an LLM agent" instead of "rows in a CRUD app" was the part that actually challenged me

at least, now, I can say that I know a thing or two about memory and LLMs

### the three types

if you think about it (very hardly, I might add), the three types map to a thing that I think is at least somewhat meaningful

`knowledge` is the corpus the agent consults but doesn't get to edit: the security KB, documentation, anything where the source is canonical and the agent's job is to look it up  
basically the stuff that's true whether or not the agent considered it and whether or not the agent has other opinions about it

`pages` is the agent's own structured durable state: things it figured out, decisions it made, notes it's keeping  
in a way, I treat them as some sort of long-term memory: mutable but with conflict detection, because long-term memory that gets clobbered silently isn't long-term memory anymore  

`journal` is the ordered log of what the agent did: append-only, replayable, no rewriting allowed  
to continue the metaphor, the short-term memory of an agent

to be completely honest, I noticed this mapping after I'd built all three, not before, but I also noticed I kinda sound smart when I say it and I like the comparisons

it's the kind of thing where the structure was driven by the operations each shape needed, and then afterwards it turned out the shapes also corresponded to my way of thinking about memory in general  
which is nice, but I want to be honest that the cognitive-model framing is post-hoc rationalization, not the thing that drove the design

what drove the design was: the operations from one shape kept contaminating the others when I tried to collapse them:  
if `pages` were append-only, they weren't long-term memory anymore  
if `journal` allowed updates, it wouldn't be a log anymore  
if `knowledge` was writable from the agent path, the corpus the agent later retrieves from could be poisoned by the agent itself, which breaks the entire reason `knowledge` collections exist

three types was the smallest split that let each shape be what it actually was

### what this costs

the obvious cost is that there are now three sets of operations to understand: when an agent needs memory, the system or the agent has to know which type fits that's a decision that doesn't exist in a single-store design

the less obvious cost is that `pages` and `journal` have different consistency stories than `knowledge`

sqlite as canonical for `pages`, sqlite for `journal` too, vector index as derived for both  
which means the operational story is "two stores in sync" instead of "one store"  
reindex paths exist, sync logic exists, the mental model is more complex

also, I haven't really validated all three at the same operational scale yet, which could mean the entire rewriting of internal workflows in their entirety

`knowledge` collections are running in production: multi-instance, real load, very little downtime  
`pages` and `journal` are built, tested, and used in development workflows but they haven't seen the kind of traffic `knowledge` has

I'm pretty sure the design is right, but "design is right" and "I have months of operational experience with this" are different things, and honestly I don't feel like I have some workflows that would justify the usage of both

there's also the case where the same content is genuinely both long-term and short-term, and in those cases I kinda went against my instinct and you write to both: `page` for the current state, `journal` for how it got there  
two writes, two stores, but they're answering different questions and combining them would just push the complexity into the agent's reasoning instead of the system's structure

## and now?

I'm so bad at closing posts, I am always trying to find the smart thing to say to end the whole thought process

honestly, I started this post wanting to talk about three collection types and ended up writing more about why I built them than what they do  
which is fine, also because I don't have enough data yet to say "yo, this small line made the whole thing 200x faster"

and here I am

`ragd` is running  
`knowledge` is doing real work  
`pages` and `journal` are sitting there, mostly waiting for the agentic workflows that'll actually use them

the only thing I can assure you is that the workflows are coming and that `ragd` will be the the memory layer underneath   
so yeah, even though I'm bad also at being consistent when it comes to posting here, I can say that there will be more technical posts (probably even longer than this)
