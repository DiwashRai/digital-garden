---
title: "Design Documents"
tags:
- atom
- infomedia
---
Reference:  [Design Docs at Google](https://www.industrialempathy.com/posts/design-docs-at-google/) |  [What is a design doc in software engineering?](https://www.youtube.com/watch?v=bgHL41e7vgI)  
Topics: [[Software Engineering]]   

---
## What is a design doc?
Design documents are relatively subjective and informal documents that the primary author of a software system or feature may write before beginning implementation of the software system or feature.  

The main purposes of a design document are the following:
- Identify design issues early on in the process where changes are still cheap.
	- ==Potentially contentious aspects of the design should be listed==
	- ==Considerations for alternative designs must be shown==
- Ensure cross-cutting conerns are considered and addressed. e.g. security, privacy, logging etc.
- Utilise knowledge of senior engineers as a way to incorporate the organisations combined experience into a design.
- Achieve consensus around a design. Allows for accurate implementation of a design in a collaborative environment as well as a way to settle possible disputes.
- Function as technical documentation about software systems and especially about design decisions made that may have been forgotten.

## Structure
- **Context and scope**
	- Summary of the landscape where the system will be built and what system actually is.
- **Goals and non-goals**
	- What is the system trying to achieve and more importantly...
	- What is the system not trying to achieve. ==Things that could be reasonable goals but are explicitly chosen to not be goals.==
- **The actual design**
	- System-context-diagram
	- APIs
	- Data storage
	- Code and pseudo-code: Should be used sparingly.
	- Degree of constraint: Constraint on the solution space. Greenfield projects allow no restrictions, whereas legacy systems enforce a multitude of constraints.
- **Alternatives considered**
	- List alternative designs that could have reasonably achieved similar outcomes
	- Focus on the trade-offs that each design makes.
	- It is fine to be succint, but balance that with the fact that ==this is the most important section in showing why the chosen design is the best.==
- **Cross-cutting concerns**
	- Ensure cross-cutting concerns have been considered.

## Length
- Shorter features or incremental improvements may be 'one-pagers'.
- Larger projects seem to be 10-20 pages.

## Lifecycle
- Creation and iteration
- Review
- Implementation and iteration
- Maintenance and learning


