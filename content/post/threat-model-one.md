---
title: "Threat Modelling: Part One"
date: 2026-04-26
draft: false
tags: ["skill", "security"]
---



I want to improve my threat modelling skills. 

So far I have read the first section of Adam Shostack's book Threat Modeling: Designing for Security. You have many threat modelling resources to choose from. I chose this resource because it is widely recognized as an excellent stand alone book covering the skill of threat modelling in detail. So far it has lived up to its reputation. Shostack breaks down threat modelling into its component parts and explains them in an orderly sequence.

As a side note, another test I used to arrive at this learning resource was to prompt several AI tools with the following question: `If you were going to recommend a single book to comprehensively learn the skill of threat modelling, what single book would you recommend? Why? Justify your choice, and compare it to other alternatives.` Gemini, Claude, and ChatGPT all yielded the same answer - I should read Shostack. That was another point in favour of this resource.

Shostack breaks the skill of threat modelling down into four questions:
1. What are we trying to build?
2. How can it break?
3. How can we fix it?
4. Did our fixes work?

Essentially you need to create a representation of your software, apply a structure set of threats (STRIDES, in the book) to your model systematically, and then identify and apply interventions to reduce the risk from identified threats. 

To answer the first question, Shostack proposes using Data Flow Diagrams (among other types of diagrams) to model your software. This seemed like a good place to start. I set out to complete 100 Data Flow Diagram drills to improve this sub-skill.

In all honesty I thought I already had adequate diagramming skills. I was wrong. I had evidently gone so long without creating a DFD that I largely had to start from scratch and re-learn the skill. So my first task was to figure out a set of drills I could use to improve this sub-skill and which had the following critieria:
1. Infinitely repeatable. I wanted to start with 100 iterations but also to leave open the possibility of additional drills and iterations.
2. Fast feedback. It would not be especially useful to just draw some diagrams and hope for the best. I needed to be able to quickly identify what I was doing right and wrong.

This is one case where AI tools can actually be useful. I landed on the following drills:
1. Given a detailed description of a system, a specific workflow or process, its data stores and external actors, create a data flow diagram. This can be repeated infinitely, by just continually feeding a prompt into any AI chatbot, and getting a list of scenarios back. Feedback? Paste the scenario description back into the same chat and ask for either a data flow diagram (to compare against the one I created) or a detailed explanation of flows between components. This is the most basic drill. Drawing simple Data Flow Diagrams to get familiar with diagram elements and their relationships.
2. Given a vague description of a system and workflow, create a data flow diagram. This is a more difficult version of the first drill, with the intention of forcing myself to think more about possible actors, processes, data flows, data stores and the relationships between them.
3. Prompt an AI chatbot to create a mermaid JS representation of a data flow diagram which contains 5-10 "logical" errors - illegal diagram states. In DFDs there are many possible illegal states: black holes, miraculous processes, external actors directly communicating, action verb labels. The list goes on. You provide an instruction to explain the errors below the diagram. Again this drill is repeatable, and you get instant feedback. Either you found the errors or you did not. This drill type was especially useful for understanding the possible error states in the diagrams. Claude was a lot better at this type of drill than Gemini or ChatGPT. 
4. Prompt an AI chatbot to create a mermaid JS representation of a data flow diagram which may or may not contain a single error. This was one of the most interesting and useful drills to complete. It forced me to look at every element and relationship in each diagram and walk through a list of possible errors. Often there were no errors but I could not know that until I had scrutinized the entire artifact. Again, the chatbot explained the error (if any) below the diagram, so I was getting instant feedback on my assessment.

Having done a hundred of these drills (in total, not each), I can now say that I am more confident that I can diagram effectively and identify errors in others' (and my own) diagrams. A pleasant side effect of the initial one hundred drills is that I now have a hundred data flow diagrams that I can use as I proceed to step two of the larger skill of threat modelling: understanding how the system can break. 

I'll explain how I learned that next step after I figure it out.