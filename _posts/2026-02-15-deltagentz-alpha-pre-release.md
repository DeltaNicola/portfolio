---
layout: single
title:  "deltagentz: the alpha pre-release"
excerpt: "the journey of rewriting everything in go"
date:   2026-02-15 15:00:00 +0200
categories: deltagentz
tags: [deltagentz, article, other]
toc: true
toc_label: "indice"
toc_icon: "heart"
toc_sticky: true
---

it has been a while since I posted the initial vision for deltagentz  
since then, the project has gone through a massive internal shift: I have moved the entire core from python to go, and it is officially in v1.0.0-alpha  

while the project remains closed-source during this pre-release phase, I wanted to walk through the current functional state of the framework and what is left on the board before the stable-release

I have also been experimenting with claude-code for this pre-release, but that will be a topic for another blog post

## goodbye python, welcome go

the decision of moving from python to go was something I have been pondering ever since I started working on this project

I opted to develop a fast proof-of-concept in python and validate my idea, and the case study against autogen still falls under this validation phase  

after that case study, I have found myself in a sort of development limbo, where a full rewrite was looking like hell but also expanding and refining the python codebase was looking like a waste of time and effort  

ultimately, I chose to finally move to go because it’s a better fit for the orchestration layer: while LLMs stay in their own domain, the framework needs to be fast, predictable, and easy to audit  

the python proof-of-concept served its purpose for the initial pattern design, but I wanted the framework to be more than the collection of scripts that it was growing into  
 
## current state of the alpha

I don't want this post to be a technical deep-dive into every choice I made along the way, but I want it to be a small _manifesto_ of its current state  

at the moment, the alpha already supports the full bottom-up stack (client → agent → node → flow), routing between nodes, and a functional tool system  
it’s not feature-complete, but it’s finally coherent: the core orchestration model is there and stable enough for me to build on

even so, there are a few key concepts I want to highlight that have followed almost every choice I made while making this migration

### composition over configuration

the most significant change in this alpha is the implementation of the functional options pattern: every major type (`RelayAgent`, `RelayNode`, `RelayFlow`, `Tool`) follows this pattern: a constructor that returns a minimal valid object, then a `WithOptions()` method that applies the functional options

this ensures that every agent or node is defined through a composition of options that you can choose from, extending its capabilities or changing its behaviour, instead of using the builder pattern it previously had

```go
writerAgent := agent.NewAgent().
    WithOptions(
        agent.WithClient(client_types.OLLAMA, "llama3", "http://localhost:11434"),
        agent.WithSystemPrompt("You are a creative writer."),
    )

writerNode := node.NewRelayNode("writer", "Writes creative content").
    WithOptions(
        node.WithAgent(writerAgent),
        node.WithAgentPrompt("Write a short story about a robot."),
    )
```

### fail fast, fail loud

the `WithOptions()` method calls validate immediately so, if something is wrong (a node without an agent, a tool without a function, a flow without a starting task) the framework panics at construction time, not at runtime

this was actually implemented because I kept misusing my own framework: having only small bits of time available during the day to work on it, I would usually end up having a misconfigured agent that would silently returns nil errors during a multi-step flow  
that was far harder to debug than a crash at startup, so I chose to implement custom errors to help me debug and pinpoint everything

errors are associated to various layers of the framework, and each layer has its own error struct embedding `BaseError`:

| Type | Layer | 
|---|---|
| `ConfigError` | Client config     |
| `ToolError` | Tool system |
| `AgentError` | Agent/LLM interaction |
| `NodeError` | Node orchestration |
| `FlowError` | Flow orchestration |
| `RouterError` | Router decision parsing |
| `ClientError` | HTTP/JSON client |

I will go into more details about them in another blog post

### zero-dependency

I know I can't guarantee this forever, but it sure has been something that has guided me throughout this pre-release  

the entire framework runs on the go standard library: the LLM client layer uses `net/http` and `encoding/json`; logging uses `log/slog`, tests use the standard `testing`  

there are no ORMs, routers, DI containers, or framework dependencies: this means you can vendor this into any project without dependency conflicts, and the code is readable without knowing any third-party APIs  

if this will change in the future, I want to go towards a low-dependencies approach, avoiding making the mistake of importing lots and lots of avoidable libraries

### layers compose upward

each layer knows about the layer below it, never above:

```
Flow → Node → Agent → Client
```

an `Agent` has no idea it's inside a `Node`  and a `Node` has no idea it's inside a `Flow`  

this ensures that you can use any layer independently: you can call `agent.Chat()` directly in a script, run a single `node.ExecuteNode()` for a one-shot task, or wire up a full multi-node `Flow`

## the future of the project

while I love its current state, I can see for myself how far from the first official release this framework actually is  
this is not an official roadmap yet, but these are the features i consider essential before publishing the stable-release and open-sourcing it

### streaming

the agent layer currently waits for the full LLM response before returning
streaming would let callers process tokens as they arrive

- **streaming `Chat()`**  
    the Ollama client already supports `"stream": true` at the protocol level  
    the plumbing from HTTP response to caller is the missing piece
- **flow-level streaming**  
    surface per-node streaming to the flow caller  
    this can be useful for UIs that want to show live agent output as a flow progresses through multiple nodes
- **streaming callbacks**  
    extend the agent's callback functions in order to let them fire on each chunk rather than once per complete response

### tool execution

the tool system handles one-shot or multiple synchronous calls  
real-world tool use is more complex

- **async tool execution**  
    tools that return a future/channel, allowing the agent to continue while a slow tool (API call, file I/O) completes in the background
- **tool result validation**  
    post-execution validation on tool outputs, similar to the existing pre-execution validator but for results
- **built-in tools**  
    a library of common tools: HTTP fetch, file read/write, shell command execution, JSON/YAML parsing  
    opt-in, not imported by default

### flow orchestration

the flow loop is sequential: one node runs at a time
more sophisticated orchestration patterns would unlock new workflow shapes

- **parallel node execution**  
    run multiple independent nodes concurrently when the delegation graph allows it (e.g., a dispatcher fans out to three specialists simultaneously, then a collector merges results)
- **consensus agent**  
    an optional agent attached to a parallel node that runs after all branches complete but before routing: produces a single consolidated result by selecting the best elements, deduplicating, and reconciling contradictions   
- **conditional branching**  
    delegation decisions based on programmatic conditions (not just LLM routing)  
    for example: route to node A if the output contains a code block, node B otherwise
- **subflows**  
    a node that encapsulates an entire inner flow  
    enables hierarchical composition: a top-level flow delegates to a "research" node that internally runs its own multi-node research flow
- **checkpointing and resumption**  
    serialize flow state (history, node iteration counts, shared memory) to disk or database  
    resume a flow from where it left off after a crash or intentional pause
- **flow events / hooks**  
    emit structured events at flow lifecycle points (flow start, node transition, router decision, flow end)  
    enables external monitoring, dashboards, and custom logging without modifying flow code

## wrapping up

deltagentz is now in v1.0.0-alpha, with its foundations finally in the right place and its core architecture finally settled  

I’m keeping it private for now because I still need to harden it, and I want the first public release to be something I can stand behind without excuses

until then, I’ll keep building it slowly, one missing piece at a time, updating this blog every once in a while