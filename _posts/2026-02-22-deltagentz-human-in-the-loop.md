---
layout: single
title:  "deltagentz: human in the loop"
excerpt: "because not everything should be left to the model"
date:   2026-02-22 01:30:00 +0200
categories: deltagentz
tags: [deltagentz, article, other]
toc: true
toc_label: "indice"
toc_icon: "heart"
toc_sticky: true
---

one of the things I kept bumping into while testing multi-agent flows is a very simple question: when does the framework hand back control?

in the current model, a flow runs from start to finish without interruption: the LLM routes, the agents produce output, and when the router finally returns `"None"`, you get a slice of messages back  
this is exactly what I want for fully automated pipelines, but it also makes a significant assumption: the agents' output is always worth forwarding to the next stage

the human-in-the-loop feature is my answer to that assumption

## the trust problem

there is an honest tension at the core of every agent framework: you are delegating decisions to a model that is, at its best, very good at producing plausible-looking output

for a lot of tasks, that is enough  
but for anything that has consequences (publishing content, executing system commands, sending a request to some APIs) "plausible-looking" is not the same as "correct", and "correct" is not the same as "approved"

the framework already has a safety valve in the iteration limits: a node that runs enough times gets silently removed from the delegation options, which at least guarantees termination  
but termination is not validation  
the flow ending does not mean the output is good

what was intentionally left out initially, as the codebase started to grow and the project started to get more and more enriched by features, became a necessary safety-measure: a way to pause the flow at a meaningful point, put the output in front of a human, and give that human real choices: continue, correct, or stop entirely

## the design

I deliberately kept the HITL feature outside the three-layer hierarchy

`RelayAgent`, `RelayNode`, and `RelayFlow` form the orchestration spine, and I did not want to pollute any of those layers with interactive I/O concerns

for that reason, `relay/human/` is a standalone package: it defines the `HumanNode` type and a pair of package-level I/O variables, and the flow layer wires it in through two new flow-level methods: `DelegateToHuman` and `AddHumanNode`

this separation means the human validation logic is testable in isolation, swappable without touching the agent or node layers, and clearly scoped to its own concern

## flow-level validation

the core type is `HumanNode`: a named checkpoint with a question, a list of pre-defined choices, and always an implicit final option for free-text input

```go
import "github.com/routedelta/agentz/relay/human"

reviewNode := human.NewHumanNode("review").WithOptions(
    human.WithQuestion("how would you like to proceed?"),
    human.WithOption(human.HumanOption{
        Kind:  human.Accept,
        Label: "looks good, continue",
    }),
    human.WithOption(human.HumanOption{
        Kind:         human.Refuse,
        RefuseAction: human.Correct,
        Label:        "needs revision",
        Message:      "please make it more concise.",
    }),
    human.WithOption(human.HumanOption{
        Kind:         human.Refuse,
        RefuseAction: human.Stop,
        Label:        "reject, stop the flow",
        Message:      "output rejected.",
    }),
)
```

wiring it into a flow is a single line after the source node:

```go
flow.NewRelayFlow().

    AddNode("writer", "writes content", 5,
        node.WithAgentOptions(
            ollamaClient(),
            agent.WithSystemPrompt("You are a creative writer."),
        ),
    ).
    DelegateToHuman("review").

    AddHumanNode(reviewNode).
    DelegatesTo("editor").

    AddNode("editor", "edits and polishes", 3,
        node.WithAgentOptions(
            ollamaClient(),
            agent.WithSystemPrompt("You are a meticulous editor."),
        ),
    ).
    WithOptions(flow.WithStartingTask("write a poem about the ocean.")).
    Run()
```

`DelegateToHuman` is mutually exclusive with `DelegatesTo` and trying to combine them panics at construction time with `FLOW_INVALID_DELEGATION`

this was a deliberate constraint: a node either delegates to the next stage or it delegates to a human, never both

### what happens at runtime

when the flow reaches the source node and its agent finishes, instead of parsing a router decision, the flow intercepts and calls `AskHuman()`:

```
══════════════════════════════════════════════════
 human validation: review
══════════════════════════════════════════════════
 how would you like to proceed?

  [1] looks good, continue
  [2] needs revision
  [3] reject, stop the flow
  [4] provide custom message
══════════════════════════════════════════════════
 your choice [1-4]:
```

the four possible outcomes:

| choice | what happens |
|---|---|
| accept | flow continues to the first `DelegatesTo` target of the human node |
| refuse + correct | source node re-runs with the option's `Message` injected as its agent prompt |
| refuse + stop | `Run()` panics immediately with `*errors.FlowError{Code: FLOW_HUMAN_REJECTED}` |
| custom message | human types the message; same effect as refuse + correct, with user-supplied text |

the "custom message" option is always available as the last numbered choice, regardless of how many options are configured

it felt kinda wrong to make free-text input something you have to setup explicitly: if you are already stopping to ask a human, you should always be able to say something the list does not cover

### the correction loop

the refuse + correct path is the one I spent the most time on

when the human selects it, the flow does not advance to the next node but it re-runs the same source node with the correction text as the agent prompt  
this is what makes it genuinely useful: the human is not just blocking the flow, they are guiding it

the correction is injected through a `pendingPromptOverride` mechanism in the flow loop: before the next iteration begins, the override is applied as the node's agent prompt, and then cleared  

this means the correction is one-shot: it runs once and is consumed, which keeps the flow state clean

### the hard stop

the refuse + stop path panics  
I know that sounds harsh, but I feel like it is the right call

when a human explicitly rejects output and chooses to abort, the correct response is not a graceful error return that some caller might ignore: it is an unambiguous interruption  
the panic surfaces as a typed `*errors.FlowError` with code `FLOW_HUMAN_REJECTED`, which can be recovered and handled like any other framework panic:

```go
defer func() {
    if r := recover(); r != nil {
        if flowErr, ok := r.(*errors.FlowError); ok {
            fmt.Printf("flow stopped by human: %s\n", flowErr.Message)
        }
    }
}()
```

### I/O and testability

the `relay/human` package exposes two package-level variables:

```go
var Input  *bufio.Reader = bufio.NewReader(os.Stdin)
var Output io.Writer     = os.Stdout
```

swapping them in tests is straightforward:

```go
human.Input  = bufio.NewReader(strings.NewReader("2\n1\n"))
human.Output = &bytes.Buffer{}
```

the reason `Input` is `*bufio.Reader` and not `io.Reader` is because a single flow run may present the human with multiple validation checkpoints, and each call to `AskHuman()` needs to read from the same position in the input stream  
using `io.Reader` directly would lose buffered state between calls while a `*bufio.Reader` keeps it

in tests, this means you can simulate a full multi-checkpoint interaction with a single string: `"2\n1\n"` means "choose option 2 on the first checkpoint, option 1 on the second"

## tool-level permission

the second mechanism is simpler and scoped to a single tool: `WithAskPermission()`

```go
deleteTool := tool.NewTool("delete_file", "deletes a file from disk").WithOptions(
    tool.WithParameter("path", "string", "file path to delete", true),
    tool.WithAskPermission(),
    tool.WithFunction(func(ctx context.Context, args map[string]any) (any, error) {
        return nil, os.Remove(args["path"].(string))
    }),
)
```

when the LLM calls this tool, execution pauses and the human is prompted for a yes/no confirmation before the function runs  

accepted answers are `"yes"` or `"y"` (case-insensitive)  
anything else returns `*errors.ToolError{Code: TOOL_PERMISSION_DENIED}` without touching the function

this is a different kind of validation than the flow-level one

the flow checkpoint is about the quality of the agent's output: you are reviewing what the model produced before it goes further  
the tool permission is about the consequence of the action: you are approving what the model is about to do in the real world

both matter, but they answer different questions

like the flow-level I/O, `tool.PermissionInput` and `tool.PermissionOutput` are package-level variables that can be swapped in tests

## what this opens up

the two mechanisms in this release are intentionally simple
stdin, stdout, one human, synchronous  

but the design has been deliberately kept open because the system needs to be hardened and tested in real and heavy workflows

### audit trail

right now, human decisions happen and are immediately consumed by the flow  

the natural next step is to record them: who approved, when, what the output was at the time of the decision, what correction was given  

this does not require changes to the core, it just requires a callback or event hook on `HumanNode` decisions that I'm yet to implement

once those decisions are recorded, you get a full audit trail of every human intervention across a flow run

### async and remote validation

the current implementation blocks the goroutine until stdin returns

this is fine for CLI tools and local scripts, but for anything running as a service it is the wrong model

an async validation interface would let the flow pause, serialize its state to a checkpoint, and wait for an external signal (a webhook, a database row update, a message queue entry) to resume

the checkpoint/resumption feature already exists in the framework and async human validation is essentially combining it with an external notification mechanism: the flow checkpoints itself, signals a reviewer, and resumes when the reviewer responds

this would also make it trivial to route validation requests to remote reviewers rather than whoever is sitting at the terminal

### multi-reviewer consensus

right now a single human decides

the logical extension is multiple humans, with a configurable policy: unanimous approval, majority, first-to-respond, ...

this maps cleanly onto the existing consensus agent model that already exists for parallel branch outputs

the same convergence logic that merges LLM outputs from concurrent branches could be adapted to merge human decisions

### role-based checkpoints

not every part of a flow needs the same kind of review

a content draft might need editorial approval, while a database write might need engineering approval, and a financial calculation might need compliance sign-off

a role-aware `HumanNode` could route the validation request to the right reviewer based on the node's type or tags

the framework already knows which node produced the output so attaching metadata to that relationship is a small step

### UI surface

the I/O abstraction is thin by design

right now it is a terminal prompt, but nothing about the design requires it to be

replacing `human.Input` and `human.Output` with something backed by a web socket, an HTTP endpoint, or a native UI component is the same substitution you already do in tests

the `HumanNode` interface does not care what is on the other end of those streams

I know that this is hugely similar to the [async and remote validation](#async-and-remote-validation) section, but I wanted to separate it because the end goals are different (or at least, they are in my head)

## what about now?

I was kinda hesitant to make this blog post cause I felt like the human-in-the-loop feature made the framework less autonomous while, in reality, it makes its execution trustworthy

in my mind there was a version of agent frameworks where the human was removed from the loop entirely, and while that version is useful for narrow, well-defined tasks where the output space is predictable it is not the right choice for anything broader

the question is not whether to include human judgment, it is where in the pipeline to place it and how to make it easy to setup

the design I landed on answers that or, at least, it tries to: a clean separation from the orchestration core, two scoped mechanisms for two different kinds of validation, and enough abstraction over I/O that the "human" can eventually be capable of making a decision
