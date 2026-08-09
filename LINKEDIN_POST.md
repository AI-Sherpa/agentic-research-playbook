Our agent fleet reproduced 363 ICML papers in three weeks and won a hackathon by 54 points over 2nd place.

The tempting story: we had a really good agent.

The real story: the agent was the smallest part of it.

What actually decided the outcome was the operating model wrapped around it. Five things, none of them exotic on their own:

1. Standing loops that survive a session restart. Ours died silently more than once, and a dead loop looks exactly like a loop that finished, until you actually check.

2. A checker that's structurally separate from the worker. Self-grading agents fail the same way self-graded exams do: you already believe your own explanation.

3. Memory with a reason attached. "X was wrong" is trivia. "X was wrong, here's why, here's how to spot it next time" is a diagnostic library.

4. Knowing what's actually being scored, not what's on the visible rubric. Some of our biggest point swings came from finding the gap between the two.

5. Treating a human's mid-project course-correction as durable state, not a comment that evaporates at the next reset.

I wrote up all five in full, with the specific incidents behind each one: [link to ai-sherpa.io write-up]

Curious what this looks like in domains outside research. Happy to compare notes.
