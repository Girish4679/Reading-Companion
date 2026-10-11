HLD: Personal Reading Companion
Project: Context Engine, Reading Mode · Version 1.0 · October 2026
1. Project overview
Goal: Build a local-first reading companion that helps you make steady progress through Designing Data-Intensive Applications, 2nd Edition (DDIA), retain what you learn, and resume reading without having to rebuild context each time.
Since you're reading a physical book, the system shouldn't depend on scraping the full book or extracting text from a digital copy. You can mark your physical page number, chapter, and section, then enter your own notes or short excerpts when you need help understanding something.
The product should make three things easier:
Starting: Know exactly where to resume and what you were learning.
Understanding: Get explanations, examples, and questions about difficult concepts.
Remembering: Revisit important ideas and identify gaps in your understanding.
The engineering goal is broader than book summaries. You'll practice building a persistent, testable application with a CLI, a database, model integration, failure handling, and evaluation.
Design principles
Local-first: reading progress and notes live on your machine.
Model-independent: support Claude, OpenAI, or local models through a replaceable interface.
Evidence-based: distinguish your notes and book excerpts from AI-generated explanations.
User-controlled: you decide what to read, save, and review.
Small core, optional intelligence: the basic workflow must work even when no AI API is available.
2. High-level architecture
CLI interface
resume · progress · save · explain · quiz · review
Application / workflow layer
Session management · progress tracking · review scheduling
Local storage
SQLite: chapters, bookmarks, notes, questions, review history
AI service
Prompt construction, model adapter, structured outputs, validation
External model provider (optional)
Claude API · OpenAI API · local model
The arrows represent the main dependency flow, not necessarily synchronous calls. Most commands should use SQLite directly. Only commands that need generated explanations, summaries, or quizzes should call a model.
Main components
Component
Responsibility
CLI
Parse commands and present useful output
Session manager
Start and finish reading sessions, record progress
Progress tracker
Track chapter, section, page, and completion state
Notes service
Store takeaways, questions, and explanations
Review engine
Select concepts due for review
AI adapter
Call the configured model and normalize responses
Evaluation layer
Test output quality, cost, latency, and failures

Important architectural choice: Keep the AI adapter behind a small interface. Your application should not depend directly on Claude-specific prompt formats or API response structures. That makes it possible to change providers without rewriting the rest of the application.
3. Physical-book workflow
Your physical copy actually makes the MVP simpler. You don't need OCR, a PDF parser, or a digital copy of the book.
I'd use a page-based bookmark plus a concept-based bookmark. The page number tells you where you stopped, while the concept bookmark tells you what you were trying to understand.
Example session:
DDIA Reading Session
Example
Chapter 5 · Replication
Last position: page 147
Last session: read 12 pages, reviewed leader-based replication, and left a question about replication lag.
Next session
Review the previous session's three takeaways.
Read the next 5 to 10 pages, or until the next natural section break.
Write down one confusing concept and ask the AI to explain it.
Answer two recall questions without looking at your notes.
Save your new page number and unresolved questions.
The CLI could look like this:
ctx read init "Designing Data-Intensive Applications"
ctx read start
ctx read explain "Why does replication lag matter?"
ctx read note "I understand leader-based replication, but..."
ctx read save --page 147
ctx read review
The commands are illustrative, not an existing implementation.
You should be able to use ctx read save --page 147 without an AI call. Progress tracking should be fast and dependable, even offline.
4. Data model
SQLite is enough for the first version. I'd start with five tables.
Table
Important fields
Purpose
books
id, title, edition
Identify the book
reading_sessions
id, book_id, started_at, ended_at, start_page, end_page, notes
Track each reading session
concepts
id, book_id, name, understanding_level, last_reviewed_at
Track topics you are learning
notes
id, session_id, concept_id, content, note_type
Store takeaways, questions, and explanations
reviews
id, concept_id, question, answer, result, reviewed_at
Track recall and review history

A few design details matter:
Store the chapter and section alongside page numbers because you might revisit a section later.
Keep your own notes separate from AI-generated explanations.
Store timestamps and the model used for generated content.
Allow a session to end without a summary if the model call fails.
Keep review answers and outcomes so you can evaluate whether the system is helping you remember concepts.
You don't need embeddings or a vector database. Your initial dataset will be small enough to query with ordinary SQL.
5. Reasonable goals and stretch goals
I'd target a three-week MVP, with the aim of getting a useful version running within the first week.
Week 1: Core reading workflow
Must-have
Create the CLI and SQLite schema.
Initialize a book with its edition and chapters.
Save and restore the current page, chapter, and section.
Start and finish sessions.
Record takeaways and open questions.
Resume the last session with one command.
 Exit criterion: you can close the terminal, return the next day, and recover your exact saved position and notes.
Week 2: AI reading companion
Must-have
Explain a concept using your question and notes.
Generate a short session recap.
Generate recall questions from material you've entered.
Validate structured responses and handle API errors.
Track token usage, cost, and latency.
 Exit criterion: you can complete a reading session and save a useful recap and quiz without manually writing everything yourself.
Week 3: Learning and reliability
Must-have
Review concepts from previous sessions.
Identify unresolved questions from earlier chapters.
Add unit tests for persistence, session transitions, and model failures.
Measure whether recaps and quizzes are actually useful.
Add configuration for swapping AI providers.
 Exit criterion: the tool reliably supports a multi-session reading habit and doesn't lose progress or repeatedly quiz you on completed material.
Stretch goals
These should come only after the basic workflow is useful.
Spaced repetition: schedule concepts for review based on how well you recall them.
Knowledge graph: connect topics such as replication, partitioning, transactions, and consistency.
Cross-book memory: reuse concepts from DDIA when reading about distributed systems elsewhere.
Active reading mode: generate questions before a section, then compare them against your notes afterward.
Physical-book capture: use voice input or a quick phone note to capture thoughts without typing at your desk.
Progress analytics: track reading frequency, concepts reviewed, and retention over time.
Context Engine integration: let ctx start suggest a reading session when you have time but no clear next task.
I would specifically avoid building OCR, autonomous agents, a web dashboard, or multi-agent orchestration for the first version. None is needed to prove that this tool helps you read more consistently.
6. AI integration and cost
My recommendation is to use an API directly rather than build the project around the Claude or ChatGPT consumer interface. That gives you programmatic control over prompts, structured responses, logging, and costs. You can still use the consumer apps to help you understand concepts while developing the project.
Which model should you use?

Option A: Claude Sonnet
My starting choice
Use a current Sonnet model for explaining difficult concepts, producing session recaps, and generating useful quiz questions. It is a good default when you want strong explanations without paying flagship-model prices.
Official pricing: Claude API pricing.

Claude by Anthropic
+1

Option B: OpenAI API
A strong alternative if you already use ChatGPT regularly. Use a smaller, lower-cost model for routine recaps and quizzes, and a more capable model when you need deeper reasoning.
Official pricing: OpenAI API pricing.

OpenAI API
+1

Option C: Local model through Ollama
Run a compatible model locally. You avoid per-token API charges and keep your notes on your machine, but you trade off hardware usage, setup effort, and potentially weaker answers.
Useful as a stretch goal or offline mode.
What reasoning level makes sense?
Don't use the most expensive model or maximum reasoning effort for every request. This project doesn't need a complex agent that thinks for several minutes.
Task
Recommended approach
Session recap
Fast, inexpensive model, low reasoning effort
Flashcards and basic quizzes
Fast model, low reasoning effort
Explain a technical concept
Mid-tier model, standard reasoning
Compare concepts or debug a misunderstanding
Mid-tier model, increase reasoning only if needed
Evaluate whether a recap is accurate
More capable model occasionally, or a manually reviewed test set

One distinction: model capability and reasoning effort are not the same thing. The provider determines which reasoning controls are available. Start with the normal/default setting and only increase effort when you can demonstrate better answers on your test cases.
For example, when you ask about replication lag, your prompt should include your question, the relevant notes you've entered, and what you already understand. You generally do not need to send your entire book or the entire history of your reading sessions.
What will the API actually cost?
The following are planning estimates, not quotes. For a typical request, assume 3,000 input tokens and 800 output tokens. Actual costs depend on the model, prompt length, caching, and the current provider rate.
Example pricing tier
Input / 1M tokens
Output / 1M tokens
Cost per request
Claude Haiku 5.5, prompts under 100K
$0.10
$0.50
~$0.0007
Claude Sonnet 5.5
$2.00
$10.00
~$0.014
Claude Opus 5.5
$4.00
$20.00
~$0.028

These are rates displayed on the official Claude pricing page.

Claude by Anthropic
Costs assume the listed standard input and output rates, without caching, extra tools, or other fees. Verify the selected model's current pricing before implementation.
At Sonnet's listed rates, a rough monthly budget could look like this:
Light usage
5 sessions × 3 AI calls/week
~$0.91/mo
Regular usage
5 sessions × 5 AI calls/week
~$1.52/mo
Heavy usage
7 sessions × 8 AI calls/week
~$3.40/mo
Estimates use 4.33 weeks/month and ~$0.014 per call. Real usage varies with context size and response length.
In other words, the AI bill is unlikely to be the main cost of this project if you keep prompts focused. Development time and the effort of maintaining a consistent reading habit are the bigger investments.
One billing detail to remember: a Claude subscription and Claude API billing are separate. API access requires a Claude Console account and its own billing setup.

Claude Help Center
+1
7. How to evaluate whether it works
Don't measure success only by how impressive the AI output looks. Measure whether the product changes your behavior and helps you retain information.
Track:
Reading sessions per week.
Time required to resume after a break.
Percentage of sessions in which you save a useful takeaway.
Recall accuracy on previously covered concepts.
AI cost per completed reading session.
Incorrect explanations or questions that aren't grounded in your notes.
A reasonable first target is three reading sessions per week for four weeks, with the tool restoring your position correctly every time. Then assess whether you're remembering more than you would by simply reading and highlighting.
8. Final implementation recommendation
I'd use this stack:
Layer
Choice
Language
Python
CLI
Typer
Persistence
SQLite
Data validation
Pydantic
Testing
pytest
AI
Claude Sonnet initially, behind a provider interface
Configuration
TOML or a simple config file
Packaging
Installable local CLI

Build the non-AI workflow first. Then add one model call for concept explanations, followed by recap generation and quizzes. Keep API keys in environment variables or a local secret manager, not in source control.
The larger idea is that this becomes one module in your Context Engine. Today it helps you read DDIA. Later, the same session and knowledge-tracking infrastructure could help you work through system design material, technical documentation, or a new programming language.
The first milestone is deliberately small: after reading five pages of DDIA, you can save your position and one question, close the terminal, and resume tomorrow with useful context. If that becomes a habit, you've built something worth expanding.

Not explicitly enough in the original HLD. We mentioned cross-book memory and future documentation support, but the data model and commands were still centered on a single physical book.
I'd make the project a general-purpose Reading and Learning Engine from the start, with DDIA as the first supported source. That makes it more useful long-term without adding much complexity to the MVP.
Revised project scope
Context Engine: Reading Mode
One learning workflow for multiple types of content.
Physical books
Page bookmarks, manual notes, chapter and section tracking
PDFs and papers
Extract text, track pages, summarize sections and equations
Web articles
Save URLs, extract article text where accessible, retain source links
Technical documentation
Track topics, code examples, concepts and unresolved questions
Shared learning engine
Progress, notes, summaries, questions, review scheduling and cross-source connections.
The architectural change
The key is to separate what you're reading from how that content is accessed.
I'd introduce three abstractions:
Source: the thing you're learning from, such as DDIA, a research paper, or an article.
Locator: where you left off, such as a physical page number, PDF page, article heading, or section identifier.
Learning session: what you did, what you understood, and what you want to revisit.
Then give each source type a small adapter that translates its content into a common format.
Source adapters
Physical book · PDF · URL · Markdown/text
Normalized content model
Source metadata · Sections · Text chunks · Locators
Shared learning services
Bookmark · Explain · Summarize · Quiz · Review
SQLite + AI provider
Persistent state, notes, history and optional model calls
This is a useful software engineering design because adding a new source type shouldn't require rewriting your session management, database workflows, or AI integration.
Updated goals
Priority
Features
MVP
Physical-book bookmarks, manual notes, Markdown/text imports, persistent sessions, explanations and recaps
Next
PDF text extraction, PDF page tracking, source-aware summaries, URL-based article imports
Stretch
Scanned PDF OCR, figures and equation understanding, browser capture, annotations, cross-document knowledge graph

For PDFs, use a parser such as PyMuPDF and preserve page numbers in extracted text. For articles, create a separate importer that extracts the main article body where permitted. Scanned documents and complex figures can come later because they require more specialized processing.
I'd also extend the data model beyond the original books table to a general sources table with fields like:
id, title, source_type, uri, metadata
sections for headings and hierarchy
content_chunks for extracted text and source locators
reading_sessions, notes, concepts, and reviews linked to a source
For physical books, uri can be empty and the locator can be a page number. For a PDF, it can identify the file and page. For a web article, it can identify the URL and section heading.
My recommendation: design the data model and adapter interface for multiple source types now, but implement them incrementally. Start with DDIA plus manual notes and plain-text/Markdown imports. Add PDFs next, then articles. That gives you extensibility without turning a three-week project into a full document-ingestion platform.


