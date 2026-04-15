## [ PNPT Overview ]

PNPT (Practical Network Penetration Tester) is a hands-on penetration testing certification created by TCM Security.

Unlike traditional certifications that rely on multiple-choice questions or simple CTF formats, PNPT is a practical exam that simulates a real corporate environment.

Rather than simply finding vulnerabilities and submitting flags, it evaluates the entire process just like a real penetration test — from external exploitation to internal pivoting, Active Directory attacks, report writing, and even a debriefing (presentation).

The exam structure is as follows:

5 days of hacking (penetration testing of a real corporate network)
2 days of report writing (submission of a professional penetration testing report)
15-minute debriefing (presenting the report and Q&A in front of an evaluator)

The passing criteria goes beyond a simple score — it includes report quality and debriefing performance, making it a comprehensive evaluation of one's abilities as a real-world penetration tester.

## [ Actual Time Required ]

Officially it's 5 days (hacking) + 2 days (report), but in reality it took about 2 weeks.

This is because after submitting the report, there's a review period of around 3 to 4 days.

Also, once the report passes, you only need to complete the debriefing within 2 weeks — so I spent about a week preparing after passing the report and then did the debriefing. (The timeline may change, so it's worth checking via email.)

## [ Preparation ]

Rather than following a dedicated study plan, I studied by consistently organizing cheat sheets on my blog.

I categorized and documented OSINT, AD attack techniques, pivoting methodologies, and more — structured in a way that I could reference during the exam as well.

In the end, this approach proved very helpful in saving time during the exam.

## [ Actual Exam Experience ]

Hacking part:

To put it simply, the attack surface for the external exploitation was smaller than expected, so it didn't feel that difficult.

The internal pivoting part was the most challenging, though obtaining the DC credentials itself was easier than I anticipated. (The key was figuring out how to move through the internal network and connect the pieces of information.)

One important tip: how well you gather information during OSINT directly determines how smoothly the later stages go.

Information missed early on often became a bottleneck later.

Report part:

The report ended up being around 47 pages.

The key to writing a good report is making it a habit to take screenshots immediately at each stage of the attack and to systematically note what commands were used and why.

I also strongly recommend preparing a report prompt (template) in advance.

Since the same template can be reused when preparing for OSCP later on, having it ready ahead of time can significantly reduce the time spent on report writing.

Debriefing part:

The debriefing was more demanding than I expected.

There are reviews online saying "the evaluator doesn't ask many questions," so I went in somewhat underprepared — but I ended up receiving 5 to 6 or more questions.

The questions generally fell into two categories:

In-depth questions about the attack techniques I used during the exam, and questions about additional attack techniques that could have been used in the same situation.

It wasn't just "what did you do" — it went as far as "why did you choose that technique and were there no other options."

Also, the most important thing when structuring the debriefing is to explain everything around "what impact this attack has on a real business."

Rather than simply saying "I accessed this system through this vulnerability," it's far more effective to explain it in terms of impact — for example, "if this attack succeeds, the potential business risk to the client would be ~."

It's also a good idea to know the alternatives to the attack techniques you used.

For example, if you used technique A in a certain situation, thinking in advance about whether technique B or C could achieve the same result will help with debriefing preparation.

Also worth noting — with a 47-page report, 15 minutes for the debriefing felt tighter than expected. I'd recommend structuring the presentation around the core attack chain and impact.

## [ Who Should Take This ]

If you're preparing for OSCP, I strongly recommend doing PNPT first without question.

The three biggest areas of growth I felt through PNPT were:

- The importance of organizing a methodology (rather than attacking impulsively, you develop a systematic process)
- Report writing ability (the skill of translating technical content into documentation is equally required in OSCP)
- Debriefing ability

Especially if you're worried about report writing or feel your methodology is still underdeveloped, I strongly encourage experiencing PNPT before tackling OSCP.
