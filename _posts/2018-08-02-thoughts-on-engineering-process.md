---
layout: post
title:  "Thoughts on engineering process"
tags: problem, solution, design, engineering
---

Engineering is solving problems. Software engineering is solving problems with software. *Professional* SWE is doing so sustainably, repeatedly, continuously. This means some kind of process is necessary.

# Understand the problem

It starts with problems. It’s very rare to see someone asking: *what problem are you trying to solve?* I personally ask it all the time. *What are the goals we want to achieve?* *What usecases do we expect?* Understanding the problem makes the next steps much easier, and you eliminate a lot of failure cases. It’s very cheap to iterate on problem statement. It’s very expensive to jump to implementation and at the end, realize that the problem was underspecified, or that you were solving the wrong problem all this time.
This applies both to high-level goals as well as to day-to-day tasks. Of course, the cost of mistake is higher in the former case.

When someone tells you to do something, and especially says it’s urgent, it’s tempting to just jump into coding. Stop. Are you sure you understand what problem they want to solve? Separate *what they want to achieve* (the problem) from *what they tell you to do* (the "solution"). Engage your critical thinking. It will often be the case that the actual thing that needs to be done is very different from what they told you originally.

Similarly, when someone asks you how to do something, it’s tempting to just offer an advice. Stop. Are you sure you understand what problem they are trying to solve? Separate *what they want to do* (the problem) from *what they ask you how to do* (the "solution”). It’s called [the XY problem](https://en.wikipedia.org/wiki/XY_problem). It will often be the case that you need that context to provide better advice.

Clarify the requirements ruthlessly. Get them to open a ticket. Bonus points if you can agree on metrics to define *how do you know* when the problem is solved.

# Design before implementation

When you understand the problem, it’s tempting to jump straight to the implementation. Stop. Think about your approach, on a high level. How can it fail? Again, engage your critical thinking: you can foresee a lot of failure cases. It’s possible that at this point you’ll realize the problem is underspecified - it’s great, go back and clarify the requirements. Consider different approaches and compare them. A little imagination goes a long way. Again, it’s cheap to iterate on the design. It’s very expensive to jump to implementation and at the end, realize that there already exists a solution, or there is a better approach.
This applies both to large long-term projects as well as day-to-day ones. Of course, the cost of mistake is higher in the former case.

Describe your approach. For larger projects, write a design doc. Iterate on it. Ask others to review it. State the motivation, goals, non-goals, limitations, and risks. Come up with alternative solutions and compare them.

If you think hard and make important design decisions at the design stage, you’ll notice that the implementation actually becomes pretty much straightforward. The explicitly written problem statement and the high-level design document will make the code review much easier. They will also be helpful when someone else many months later will try to understand why things are implemented the way they are.

# Bottom line
For smaller tasks, the temptation to shortcut the process is higher. It’s understandable; at a minimum, use the ticket to iterate on the problem statement and the approach. At the end of the day, it’s the result that counts. And the result is that you’ll be solving exactly the problem you want to solve, using the most suitable approach that you chose consciously.

Always ask what problem are we actually trying to solve. Don’t waste time solving the wrong problem. Don't overengineer. Think about failure cases. Ship reliable software.
