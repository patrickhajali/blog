---
title: fcps recap
published: false
---
Modular programming, as a software design technique, is extremely widespread: it is an inductive bias embedded within the design of virtually all modern programming languages. By compartmentalizing functionality into discrete, interchangeable modules, the development process is streamlined (division of labor; reusability of code) and debugging is simplified (straightforward traceback analysis). Automated approaches to program synthesis should favor modularity, especially when generating large programs and/or implementing complex logic. 

General purpose generative models trained on a wide corpus of natural language (like transformer-decoders) have demonstrated the highest capability, to-date, at generating useful programs. However, the autoregressive decoding mechanism inherent to the these generative methods does not, on its own, promote modularity. 

It is reasonable to assume the models' training corpus contains well-designed code from which the models should implicitly learn positive design techniques. But what if the generation process itself is flawed? For one, the probability of breaking a program scales with the program length under standard autoregressive decoding (each token is a potential bug). Even worse, the models' difficulties in long-term reasoning often leads to a lack of coherence in longer generations, making it challenging to maintain consistent logic and structure throughout a program. Without an inherent mechanism to enforce modularity and structural integrity, the generated code can become unwieldy and error-prone, especially in scenarios where maintaining state or executing a sequence of dependent operations is crucial. 

In order to extend these models' capabilities in generating/discovering complex programs, code-generation systems should

1. Draw on knowledge gained from experience, like code generated as a solution to similar problems
2. Break down complex tasks into smaller, more manageable sub-tasks, and then synthesize and combine these individual solutions to construct the complete program

In the course of my Master's research at Cambridge, I developed a technique ([Function-constrained Program Synthesis](https://arxiv.org/abs/2311.15500)) designed to imbue code-generating models with the aforementioned attributes. My approach addresses the limitations of traditional approaches by contextually constraining code-generation to an explicit function set (initially defined by the user) and enabling recovery from failed attempts through automatically generated sub-functions (enforcing modularity) which can be re-used across similar tasks. 

I was inspired by a vision: the creation of a singular foundation model capable of tackling any computer vision task. This idea, while highly ambitious even now, aligns with the historical trajectory of the field. Initially, classical CV methods focused on engineering specific algorithms for distinct tasks. The advent of deep learning shifted this paradigm, offering more generalized model architectures capable of addressing a variety of tasks. This evolution was further exemplified by the emergence of foundation models like  [CLIP](https://arxiv.org/abs/2103.00020), which demonstrated proficiency across a spectrum of CV challenges. Each developmental phase effectively abstracted complex problems into increasingly capable models.

Observing the growing effectiveness of language models, I hypothesized that the next evolution for an 'all-encompassing' CV model could operate in the medium of code. Such a model, armed with the right programming tools and functions, could potentially write its own algorithms—be they classical, deep learning-based, or entirely novel approaches—to solve any given task. 

This ambitious hypothesis steered and fueled my research towards enhancing the code-generation capabilities of language models. 

### What I Learned

; presented at the 2023 R0-Fomo Workshop at NeurIPS
From the start, I held full responsibility for the research project - from the initial focus, to experiment design/execution, writing, presenting. Not only did I learn all the technical skills (become fluent with generative models, how to conduct research) and specific skills of writing/presenting a paper, the biggest learning experience is that perhaps the most difficult part of research is deciding how to approach solving a huge task.

Now, having graduated, this question has been broadened. I resonate with  I want to extend my domain and skillset by working in biotech (need to be more specific). I realize that the greatest breakthroughs come from existing across disciplines - this is what I want. Example is viewing program synthesis as natural language, this was not trivial until recently. Also evolutionary programs (FunSearch). 

While working on my research, I was reading Albert's *Molecular Biology of the Cell*. 

Role and Growth and Learning 

Why Apprenticeship Program (next step)
transition into a new field, grow my skills, want to be a generalist, very interested in the human body. 



--
In at most 750 words, tell us about something that you’ve done that you’re passionate about. 

1. What was it that really excited you about this task?  

2. What was your role in the big picture?  

3. What did you learn and/or conclude from the experience?  

4. How did you personally and/or professionally grow from the experience?  

5. What motivated you to apply to the Apprenticeship program and what do you hope to gain from it? What are ways you hope to grow your career in an antedisciplinary manner?
-- 


Logistics: came up with the idea, designing experiments, running them, writing the thesis/paper, presenting the paper 

**Why I am excited about this task**

What is the vision? LLMs are impressive. Generate very complex programs, robustly. Time to useful model is decreased. Can apply to many different areas. 

The vision we had when starting the problem is coming into fruition. LLMs are most useful if they can use tools. Functions are tools. Can they create a tool in one-step and use it in the next? 

FunSearch (mathematical discovery)

**Learning and growth (personal and professional)**
- research process
- deciding what to work on is the hardest part 

Need to think from first-principles. Programs were originally treated separately than natural language, but formatting the problem in this way changed everything. 


This bias underscores a fundamental principle in software engineering: the division of !complex systems into smaller, manageable, and reusable components.


We are taught to write computer programs in a modular manner. To implement complex programs, we introduced object oriented design. 

Most programming paradigms revolve incorporate this concept. The languages we use today have this built in as an inductive bias. 


evolutionary algorithms



Generate complicated algorithms. 
Generative AI models - frontier LLMs (GPT-4, CodeLlama-2) - 

It took time for programmers to view program synthesis as a natural language task. 

You want to go where a question takes you, not where your training left you