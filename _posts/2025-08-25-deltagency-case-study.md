---
layout: single
title:  "case study: deltagentz vs autogen"
excerpt: "comparative analysis against autogen AI agents"
date:   2025-08-25 9:00:00 +0200
categories: deltagentz
tags: [deltagentz, casestudy]
toc: true
toc_label: "indice"
toc_icon: "heart"
toc_sticky: true
---
the content creation factory showdown

***NOTE:***
*this page refers to a very early-stage development phase of the framework and it does not reflect its current state or its full capabilities*
{: .notice--info}

## overview
this case study compares a task-first, ollama-native framework (deltagentz) with microsoft's autogen under identical conditions.  

deltagentz focuses on deterministic, task-based workflows with a lightweight, zero-bloat design and a relay pattern for orchestration, while autogen emphasizes conversational collaboration.  

the analysis examines code complexity, resource management, execution patterns, and output quality to reveal unexpected insights about framework trade-offs and challenge common assumptions about enterprise versus custom solutions.

## test structure

### conditions
- **same model:** llama3.1:8b via runpod proxy
- **same agents:** researcher, writer, editor, seo_optimizer, publisher  
- **same system prompts:** identical specialization instructions
- **same task:** "create comprehensive blog post about 'ai automation trends in small businesses'"
- **same infrastructure:** same runpod gpu backend

### key differences
- **deltagentz:** relay pattern configured as a sequential pipeline for this test (researcher→writer→editor→seo→publisher); the framework also supports consultation loops and routing patterns
- **autogen:** round robin group-chat with max_turns=5

### agents and task definitions

- **model:** llama3.1:8b
- **researcher system prompt:** "You are an expert content researcher. Your job is to gather comprehensive, accurate information on given topics. Focus on finding credible sources, key statistics, trending angles, and unique insights that will make content stand out."
- **writer system prompt:** "You are a skilled content writer. Transform research into engaging, well-structured content. Create compelling headlines, smooth transitions, and maintain consistent tone. Focus on readability while preserving all important information from research."
- **editor system prompt:** "You are a professional content editor. Review content for grammar, clarity, flow, and structure. Remove redundancy, improve readability, ensure logical organization, and maintain consistent style. Make the content polished and professional."
- **seo optimizer system prompt:** "You are an SEO content specialist. Optimize content for search engines while maintaining readability. Add relevant keywords naturally, improve meta descriptions, suggest internal linking opportunities, and ensure content follows SEO best practices."
- **publisher system prompt:** "You are a content publishing specialist. Format content for final publication, add appropriate call-to-actions, ensure proper formatting for the target platform, and create engaging social media snippets for promotion. Give the final polished document as result."
- **task:** "Create a comprehensive blog post about 'AI automation trends in small businesses' that will rank well on search engines and drive engagement"

### outputs

the outputs have been copied as-is.
{% include case_study_results_displayer.html project="deltagentz" date="25-08-2025" %}

## architectural comparison

### implementation

#### deltagentz
```python
# agent definition
researcher = (ExecutorAgent(MODEL, BASE_URL, RESEARCHER, desc)
           .with_system_prompt(RESEARCHER_SYSTEM_PROMPT))

# workflow definition
deltagentz_relayer = (Relayer()
    .add_node(researcher, delegate_to=[writer])
    .add_node(writer, delegate_to=[editor])
    .add_node(editor, delegate_to=[seo_optimizer]) 
    .add_node(seo_optimizer, delegate_to=[publisher])
    .add_node(publisher, unload_after_use=True))

# task execution
deltagentz_relayer.run(TASK)
```
#### autogen
```python
# model client definition
model_client = OllamaChatCompletionClient(model=MODEL, host=BASE_URL)

# agent definition
researcher = AssistantAgent(name=RESEARCHER, 
                          model_client=model_client,
                          system_message=RESEARCHER_SYSTEM_PROMPT)

# workflow definition
autogen_chat = RoundRobinGroupChat([researcher, writer, editor, 
                                  seo_optimizer, publisher], max_turns=5)

# task execution
await Console(autogen_chat.run_stream(task=TASK))
```
### philosophy

#### deltagentz
- **clear separation of concerns** – each agent has one job
- **controlled context handoff** – agents pass the necessary state/results; relay pattern includes a clean step to release context
- **unload optimization** – final agent can unload the model to free memory
- **linear debugging** – easy to trace issues to a specific agent

#### autogen
- **collaborative iteration** – agents can respond to and critique each other
- **max_turns limit** – prevents infinite loops (set to 5 in this test)
- **async-friendly execution** – often run with an event loop for streaming; synchronous paths also exist
- **distributed decision making** – flow is powerful but less predictable

### why deltagentz differs (at a glance)
- task-focused deterministic workflows
- modular, composable agents
- ollama-native direct api integration
- lightweight runtime with minimal deps
- developer-friendly clean api

## comprehensive performance comparison

### workflow analysis

both frameworks follow the same 5 stages, but with different approaches.

| stage        | deltagentz (sequential evolution)                         | autogen (conversational collaboration)         |
|--------------|------------------------------------------------------------|-----------------------------------------------|
| **research** | collects statistics and trends, builds a solid foundation  | produces a comprehensive draft with statistics |
| **writing**  | expands with challenges, opportunities, and implementation | adds strengths/weaknesses analysis             |
| **editing**  | enriches with case studies and real-world examples         | focuses on readability (shorter, clearer text) |
| **seo**      | adds best practices, keyword optimization, structured flow | adds hooks, active voice,  breaks long paragraphs |
| **publishing** | final polish with faqs, cta, and social snippets; publication-ready formatting | suggests cta and social sharing; requires manual finalization for publication |

### execution results 

| aspect              | deltagentz                                                  | autogen                                             |
|---------------------|-------------------------------------------------------------|-----------------------------------------------------|
| **style**           | more articulated, like a long-form blog post                | more schematic, concise, and seo-optimized          |
| **real data**       | yes, includes statistics and concrete case studies          | no, only generic references                         |
| **readability**     | dense, requires focused reading                             | smooth and more immediate                           |
| **seo**             | meta + keyword density + suggested images                   | meta + headers + keywords, but lacks visuals        |
| **cta/engagement**  | strong final cta                                            | suggested to add cta/social sharing                 |
| **content depth**   | provides best practices, challenges, opportunities          | focuses more on trends and recommendations          |
| **practical tips**  | general recommendations at the end                          | actionable step-by-step tips (e.g., "start small")  |
| **examples/case studies** | 2 specific business examples with measurable results | lacks concrete examples, more theoretical           |
| **engagement strategy** | cta + faqs + image suggestions                          | suggestions to add infographics and sharing options |
| **target audience fit** | best for readers seeking in-depth analysis and authority | best for readers who want a quick, digestible overview |
| **overall use case** | ideal as a **reference article** or **pillar content**     | ideal as a **fast seo blog** or **linkedin article** |

### execution metrics

| metric | deltagentz | autogen | winner | margin |
|--------|------------|---------|---------|---------|
| **execution time** | 69.37s | 57.42s | autogen | -17% faster |
| **words/second** | 12.61 | 17.41 | autogen | +~5 words/second |
| **total words** | ~875 | ~1000 | autogen | +~12.5% more content |
| **sections completed** | 9 | 6 | deltagentz | +50% more sections |
| **statistical citations** | 4 specific | 2 generic | deltagentz | 2x more data |

### process characteristics

| aspect | deltagentz | autogen | winner | advantage |
|--------|------------|---------|---------|-----------|
| **predictability** | deterministic | variable | deltagentz | consistent output |
| **adaptability** | rigid | flexible | autogen | dynamic adjustment |
| **debugging ease** | simple | complex | deltagentz | linear troubleshooting |
| **human-like process** | mechanical | conversational | autogen | natural collaboration |
| **scalability** | excellent | moderate | deltagentz | easy agent addition |
| **learning curve** | steep | gentle | autogen | faster onboarding |

### content quality matrix

| element              | deltagentz score        | autogen score          | winner     | notes                        |
|----------------------|-------------------------|------------------------|------------|------------------------------|
| **depth of research** | very strong             | moderate               | deltagentz | concrete stats vs generic    |
| **professional polish** | highly polished        | basic                  | deltagentz | enterprise-ready             |
| **readability**      | good but dense          | smooth and accessible  | autogen    | more conversational tone     |
| **seo optimization** | advanced and precise    | limited                | deltagentz | calculated precision         |
| **practical examples** | rich and quantified    | scarce and abstract    | deltagentz | quantified case studies      |
| **engagement factors** | strong (cta + faq + social) | moderate (suggestions only) | deltagentz | complete engagement features |

### workflow efficiency 

| process aspect | deltagentz | autogen | winner | reasoning |
|----------------|------------|---------|---------|-----------|
| **setup complexity** | minimal | moderate | deltagentz | env vars vs complex config |
| **configuration effort** | low | high | deltagentz | system prompts vs framework setup |
| **maintenance overhead** | very low | moderate | deltagentz | simple codebase vs framework updates |
| **quality consistency** | excellent | variable | deltagentz | deterministic pipeline |
| **customization flexibility** | excellent | limited | deltagentz | direct prompt control |
| **team collaboration** | individual | group | autogen | natural peer review |
| **cost efficiency** | free | expensive | deltagentz | local models vs api costs |

## limitations

### disclaimer

it is important to note that deltagentz is still in a super beta stage.  

this means the results presented here should be read as exploratory and not as definitive benchmarks. several factors limit the generalizability of the comparison:

- the framework is under active development and not yet open-sourced; results reflect a snapshot before production stability
- the framework has not yet been optimized for performance or resource efficiency  
- results are based on a single content generation task and may vary in other domains  
- debugging and logging features are still minimal, which may affect reliability in larger tests  
- no external validation has been performed, so conclusions are based solely on this controlled experiment  

these limitations highlight that while the comparison offers useful insights into tendencies and trade-offs, the findings should not be taken as final evidence of superiority in production environments. instead, they provide a snapshot of how the two frameworks behave under equal conditions at this stage of development.

### honest autogen advantages

autogen shows clear strengths in several areas that are worth acknowledging:

- **peer review process:** agents critique and refine each other’s output  
- **enterprise integration:** natural fit for teams already in the microsoft ecosystem  
- **conversational development:** interaction style may feel more natural to some teams  
- **cloud scalability:** strong option for organizations with sufficient budget for token costs  

### setup complexity comparison

| aspect | deltagentz | autogen (ref impl.)  | winner | reality |
|--------|----------------------|--------------------------|--------|---------|
| configuration | model and base url passed in code | model and base url passed in code | draw | both can read from env or constants |
| code required | compact pipeline: executoragent + relayer + run | compact pipeline: client + agents + groupchat + run | draw | similar amount of code |
| dependency management | deltagentz runtime + model backend | autogen runtime + model backend | draw | both need a client lib and an ollama-compatible backend |
| deployment complexity | needs a reachable model backend | needs a reachable model backend | draw | identical requirement at this level |
| customization effort | edit per-node system prompts | edit per-agent system messages | draw | prompt-first customization on both |
| debugging difficulty | linear node chain via relayer | intertwined group chat turns | deltagentz | easier causal tracing from node to node |
| context handling | full history forwarded downstream | shared conversation state across agents | draw | different mechanisms, comparable effect |
| resource control | explicit `unload_after_use` on final node | managed by framework runtime | deltagentz | explicit release hint present |
| async requirements | not required | event loop and streaming console | deltagentz | fewer runtime prerequisites |

## discussion

### quick comparison

the comparison highlights two distinct philosophies with clear trade-offs:

- **output style**  
  - deltagentz: long-form, structured, data-rich, enterprise-like  
  - autogen: shorter, smoother, more conversational and more readable 

- **workflow behavior**  
  - deltagentz: sequential, deterministic, easy to trace  
  - autogen: iterative, collaborative, harder to predict  

- **strengths**  
  - deltagentz: depth, concrete examples, stronger seo structure  
  - autogen: readability, speed, natural peer-like review  

- **weaknesses**  
  - deltagentz: rigid when configured as a strict sequential pipeline; consultation loops were not enabled in this test 
  - autogen: shallow on data, fewer concrete case studies  

- **best fit use cases**  
  - deltagentz: reference articles, pillar content, enterprise documentation  
  - autogen: brainstorming drafts, seo blogs, fast social content  

### qualitative analysis
overall, deltagentz aligns with a production-first task-based mindset, while autogen reflects a creative and exploratory approach. the choice depends less on raw performance and more on the intended content type and context of use.

#### deltagentz vertical specialization
**pros:**  
- each agent maintains specific expertise  
- incremental quality output  
- highly professional final result  
- deterministic process control  

**cons:**  
- less flexibility during execution  
- no feedback between previous phases  
- more rigid process  

#### autogen horizontal collaboration
**pros:**  
- more human and reflective process  
- integrated self-correction  
- dynamic adaptability  
- proactive problem identification  

**cons:**  
- dispersion of specialized competencies  
- communication overhead  
- less thorough results  
- loss of deterministic control  

## personal reflections

after running both frameworks side by side, a few personal takeaways stood out:

- **surprises**  
  - as expected, autogen produced more volume, but deltagentz generated richer content  
  - the relay pattern in deltagentz felt more predictable than anticipated, close to a production pipeline   

- **frustrations**  
  - overall it is kinda impossible to not notice how quick it is to plug & play autogen once you know your way around it  
  - with the strict sequential configuration used here, deltagentz left little room for agents to challenge earlier steps  

- **satisfactions**  
  - seeing deltagentz deliver professional, publication-ready structure in a beta stage was encouraging  
  - autogen’s conversational dynamics yielded drafts that needed minimal copy editing  

- **future tests**  
  - enable consultation loops and routing in deltagentz to test adaptability  
  - profile resource usage and latency across longer documents and different models  
  - evaluate hybrid pipelines that inject targeted peer-review stages  
  - replicate the experiment across multiple topics and audiences to check generalizability  

these reflections underline that the experiment was less about finding a winner and more about understanding how different philosophies of agent collaboration shape the final result

## conclusion
- **deltagentz**  
  - task-based with strict role separation  
  - deterministic flow ensures traceability and consistent quality  
  - best fit for structured tasks, long-form content, and enterprise use  
  - excels when predictability and accountability matter most 

- **autogen**  
  - strong in readability and speed  
  - collaborative iteration feels natural, almost like peer review  
  - best fit for brainstorming, quick drafts, and exploratory work  
  - excels when adaptability matters more than precision  

in essence: autogen is conversation-first and favors flexible collaboration; deltagentz is task-first and relay-driven, favoring controlled execution with predictable outcomes.

<br/>
---
<br/>

*this was honestly quite a journey.*  

*mainly because I had never written something like this before, nor had I run these types of tests with so many different factors in mind.*   

*i hope to bring more case studies in the future, as I enrich deltagentz with more and more features before releasing it and open-sourcing it.*