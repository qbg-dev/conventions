# One-Pager Conventions

Rules for writing one-pagers that get stakeholders aligned on a problem, the approaches being considered, and what's needed from them. Derived from the Amazon 6-pager discipline (narrative prose, every sentence carries information, writing is thinking) compressed to one page.

A one-pager is not a PRD, not a pitch deck script, not a status update. It is a considerate, thoughtful overview of a specific situation written for people whose time you respect.

A good one-pager has three sections:

```
1. BACKGROUND    — What's happening, concretely. One paragraph.
2. APPROACHES    — How we're thinking about solving it. 2–3 options with tradeoffs.
3. ASKS          — What we need from you. Specific questions, decisions, information.
```

---

## Structure (rules 1–15)

1. **Background is one paragraph, 4–6 sentences.** It answers: what is the situation, who is affected, and why does it matter now. Not why it matters in general—why it matters this week.

2. **Background names concrete things.** Systems, clients, dollar amounts, dates, people. "Baozheng's property management platform processes 12,000 maintenance requests/month through WeChat" not "our client has a high-volume service operation."

3. **Background does not explain the category.** If the reader needs "AI agents are software that..." you're writing for the wrong audience. Start from what they already know.

4. **Background ends with the problem statement.** The last sentence of the paragraph is the thing that needs solving. Everything before it is context for understanding that sentence.

5. **Approaches section presents 2–3 options.** Each option gets a short paragraph: what it is, why it might work, what the tradeoff is. If there's only one option, you don't need a one-pager—you need approval.

6. **Each approach has a tradeoff, not just benefits.** "Fine-tuning gives us domain accuracy but requires 2 weeks of labeled data we don't have yet" not "Fine-tuning gives us superior domain accuracy."

7. **Approaches are at the right altitude.** High enough for a non-technical stakeholder to follow, specific enough that an engineer reading it knows what you mean. "RAG over their existing Confluence docs" not "retrieval-augmented generation leveraging enterprise knowledge bases."

8. **State which approach you recommend and why.** Don't present options neutrally and wait—stakeholders want your judgment. "We recommend option B because it ships in 2 weeks and unblocks the pilot" is a sentence that moves things forward.

9. **Asks section is a numbered list.** This is the one place where list format is better than prose. Each ask is a specific question, decision, or piece of information you need.

10. **Each ask has enough context to be answerable.** Not "What's the timeline?" but "We need to know whether the Baozheng pilot must launch before Chinese New Year (Jan 29) because that changes which approach is feasible."

11. **Asks are addressed to someone.** If you know who can answer, name them. If you don't, say so—"We need someone from finance to confirm X" is better than a question floating in space.

12. **Target 1–4 pages depending on complexity.** Simple alignment: 1 page. Multi-stakeholder decision with several approaches: 2–3 pages. Complex situation with significant context: 4 pages max. If it exceeds 4, you're either writing a different document or haven't done the hard work of deciding what matters.

13. **No sections beyond the three.** No executive summary (it's one page—the whole thing is the summary). No appendix. No "next steps" (that's what the asks section is). No conclusion.

14. **Title is the problem statement, not the project name.** "Should we build a custom OCR pipeline or use Azure Document Intelligence for Baozheng invoices?" not "Baozheng AI Integration Project Overview."

15. **Date it.** One-pagers age fast. Put the date under the title so readers know if the context has shifted.

---

## Writing rules (rules 16–35)

16. **Every sentence must contain information the reader didn't have before reading it.** If you delete a sentence and the document loses nothing, delete the sentence.

17. **No repetition.** If you stated a fact in the background, do not restate it in the approaches section. The reader has short-term memory. Trust it.

18. **Narrative prose in background and approaches.** Full sentences, connected by logic ("because," "so," "which means"), not bullet points. Bullets hide incomplete thinking—prose forces you to articulate the causal chain.

19. **Be honest about what you know vs. what you don't.** "We don't have latency numbers yet—that's ask #3" is more useful than an estimate you made up. Gaps are information. Fake precision is noise.

20. **Don't manufacture data to sound rigorous.** Early-stage one-pagers often lack metrics. That's fine. State what you've observed, what you've heard from the client, what you're assuming. Label assumptions as assumptions.

21. **Use specific names, not roles.** "Matt tested this on the staging environment" not "our CTO validated the technical approach." Names create accountability and make the document feel real.

22. **Use actual numbers when you have them. Don't invent them when you don't.** "Baozheng processes ~12,000 tickets/month" (you counted) is good. "This could save up to $500K annually" (you didn't count) is consulting-deck filler.

23. **Write in present tense for current state, past tense for what happened.** Not future tense for things you hope will happen. "The current system drops 15% of requests" not "the new system will reduce dropped requests by 15%."

24. **One idea per sentence.** Compound sentences with three clauses connected by "and" are three sentences pretending to be one. Split them.

25. **No weasel words.** "Significant," "substantial," "considerable," "robust"—these mean nothing without a number or comparison. Replace with specifics or delete.

26. **No hedge stacking.** "We might potentially be able to possibly" → "We can, if X." One hedge maximum per sentence.

27. **No passive voice for decisions.** "It was decided that..." → "We decided..." or "Sarah decided..." Decisions have owners.

28. **Paragraphs are 3–5 sentences.** Longer paragraphs lose the reader. Shorter ones feel like bullets wearing paragraph costumes.

29. **Transitions are logical, not verbal.** Connect paragraphs through the actual relationship between ideas, not with "Furthermore," "Additionally," "Moreover." Those words are duct tape over bad structure.

30. **Define abbreviations once, then use them.** "Baozheng Property Management (BPM)" then "BPM" throughout. Don't spell it out every time—that's repetition.

31. **Round numbers unless precision matters.** "~12,000 requests" not "11,847 requests" unless the exact count is the point.

32. **Time references are absolute.** "By January 29, 2026" not "within the next 6 weeks." The reader might read this document three weeks from now.

33. **Technical terms are fine if your audience knows them. Jargon is not.** "RAG pipeline" is a technical term your engineering stakeholder knows. "Synergistic knowledge retrieval framework" is jargon nobody needs.

34. **The writing is the thinking.** If the one-pager is unclear, the strategy is unclear. Don't polish unclear thinking—rethink it.

35. **Write it, leave it overnight, then cut 30%.** First drafts contain the thinking process. The final draft contains only the conclusions.

---

## What to include (rules 36–50)

36. **The constraint that makes this hard.** Every interesting problem has a constraint. "We need to launch before Chinese New Year AND support Mandarin voice input" is more useful than describing either requirement alone.

37. **What has already been tried or ruled out, and why.** "We evaluated Azure's off-the-shelf OCR but it misreads 40% of Chinese receipt formats" saves the reader from suggesting it.

38. **Dependencies and blockers.** If approach B requires API access that takes 3 weeks to provision, say so in the approach description, not as a surprise in the asks.

39. **Cost and timeline for each approach.** Even rough estimates ("2 weeks and ~$3K in API costs" vs. "6 weeks and ~$15K in contractor time"). If you genuinely don't know, that's an ask.

40. **What you're assuming.** "We're assuming Baozheng can provide 500 labeled invoices for training" makes an implicit dependency explicit. If the assumption is wrong, the reader can flag it immediately.

41. **Who else has context.** "Kevin from Baozheng's IT team has the API docs and has offered to help with integration testing." This helps stakeholders know who to loop in.

42. **The risk you're most worried about.** Not a risk matrix—one sentence about the thing that keeps you up at night. "The main risk is that their legacy system has no API and we'd need to screen-scrape, which is fragile."

43. **What success looks like.** One sentence. "Success is Baozheng processing invoices with <5% error rate by end of Q1." Not a paragraph—a sentence.

44. **The deadline and why it exists.** "Chinese New Year is Jan 29; Baozheng's office closes for 2 weeks, so anything not deployed by Jan 22 waits until mid-February."

45. **What you've already done.** Two sentences max. "We've run a proof-of-concept with 50 sample invoices and achieved 92% accuracy with GPT-4V. The remaining 8% are handwritten annotations." This builds credibility for your recommendation.

46. **Quoted stakeholder language if it clarifies the problem.** "Their CEO said 'we lose 2 hours per person per day on manual data entry'" is more vivid than "they report significant time spent on manual processes."

47. **Comparison to similar past work.** "This is similar to the invoice pipeline we built for Chen Properties, which took 3 weeks" helps stakeholders calibrate.

48. **The thing you'd ask if you were the reader.** Anticipate the obvious question and answer it preemptively.

49. **Links to live demos, prototypes, or screenshots—if they exist.** One link, not five. The reader clicks or they don't.

50. **Your confidence level.** "We're fairly confident in approach B but want to validate the Mandarin voice input before committing." Honest uncertainty is professional.

---

## What to exclude (rules 51–70)

51. **No category explanations.** "AI is transforming how businesses..." → delete. Your stakeholders know what AI is.

52. **No history of the technology.** Nobody needs to know that transformers were invented in 2017 to decide on a vendor.

53. **No impressive-sounding details that don't affect the decision.** "Using a 70B parameter model with 4-bit quantization" doesn't help a business stakeholder choose between approaches. If it matters, explain why it matters.

54. **No generic benefits.** "This will increase efficiency and reduce costs" is true of almost anything. State the specific efficiency gain or don't mention it.

55. **No competitive analysis.** "Unlike competitors who..." is a pitch deck move. The reader isn't choosing between you and a competitor—they're choosing between approaches.

56. **No restating the problem in different words.** If the background covers it, the approaches section doesn't re-explain it. If you find yourself writing "as mentioned above," delete everything between the two mentions.

57. **No aspirational language.** "This positions us to..." "This lays the groundwork for..." "This creates a foundation for..." → all filler. State what it does, not what it "positions you for."

58. **No buzzwords as load-bearing structure.** "Leveraging our AI capabilities to deliver scalable solutions" says nothing. Replace every buzzword with what it actually means or delete the sentence.

59. **No "in order to."** "In order to" is always replaceable with "to." Delete two words, lose zero meaning.

60. **No methodology descriptions.** "Using an agile approach with two-week sprints" is how you work, not what the stakeholder needs to know. They need the timeline and deliverables.

61. **No org chart or team structure.** Unless a stakeholder specifically asked who's doing what. Even then, one sentence: "Matt leads engineering, Warren handles client relationship."

62. **No risk matrices.** A 3x3 grid of likelihood vs. impact is theater. Name the one risk that matters (rule 42) and move on.

63. **No assumptions section.** Don't list assumptions separately—weave them into the relevant approach. "This assumes Baozheng provides API access by Jan 10" lives in the approach that depends on it.

64. **No success metrics section.** One sentence in the body (rule 43) is sufficient. A separate KPI section is a PRD move.

65. **No glossary.** If you need a glossary, you're using too much jargon.

66. **No version history.** Date the document (rule 15). If it changes substantially, write a new one.

67. **No disclaimers.** "This is a preliminary assessment and subject to change" applies to everything humans write. Don't say it.

68. **No "please don't hesitate to reach out."** This is a one-pager, not a cover letter.

69. **No logos, headers, or branding beyond the title.** Content, not decoration.

70. **No content that exists only because a template told you to include it.** If a section doesn't serve the reader for this specific document, cut it. Templates are starting points, not obligations.

---

## Tone (rules 71–85)

71. **Write like you're emailing someone busy who trusts your judgment.** Not a board presentation, not a blog post, not a proposal to a stranger. An internal email to someone who will spend 3 minutes on this.

72. **Professional means clear, not formal.** "We think option B is the move" is fine. "It is our considered recommendation that option B represents the optimal path forward" is not.

73. **State your opinion.** The reader wants your take. "We should do B" is more useful than "There are several options to consider." You've done the thinking—share the conclusion.

74. **Confidence without arrogance.** "We recommend X because Y" not "X is clearly the only viable option." Leave room for the stakeholder to disagree with grace.

75. **High-level, not hand-wavy.** High-level means you've compressed the details into clear conclusions. Hand-wavy means you haven't done the work yet. The reader can tell the difference.

76. **No performative complexity.** Don't make something sound harder than it is to justify resources. Don't make something sound easier than it is to get approval. State the difficulty honestly.

77. **No deference theater.** "We wanted to get your thoughts on..." → "We need a decision on X by Friday." Respect the reader's time by being direct about what you need.

78. **No consulting-deck voice.** If it sounds like McKinsey wrote it, rewrite it. Consulting decks are designed to justify fees. One-pagers are designed to move work forward.

79. **Use "we" for the team, "I" for personal observations.** Not "one might consider" or "it could be argued."

80. **Match the reader's vocabulary.** If the stakeholder says "WeChat mini-program," don't upgrade it to "mobile application ecosystem." Use their words.

81. **No urgency inflation.** Everything is "critical" and "time-sensitive" in bad documents. If the deadline is real, state the date and the consequence of missing it. The reader will feel the urgency from the facts.

82. **Acknowledge complexity without drowning in it.** "The integration has three hard parts: auth, data mapping, and Mandarin NLP. We've solved auth, data mapping is straightforward, and Mandarin NLP is the open question." That's enough.

83. **If the situation is bad, say so plainly.** "We're behind schedule. The OCR accuracy is 72%, not the 95% we need. Here's what we're doing about it." Stakeholders respect honesty more than spin.

84. **Read it aloud. If it sounds like a consultant, rewrite it. If it sounds like a smart colleague explaining something at a whiteboard, ship it.**

85. **The best one-pager is the one that makes the meeting 15 minutes instead of 60.** If the reader finishes with a clear picture, clear options, and clear asks, you've succeeded.
