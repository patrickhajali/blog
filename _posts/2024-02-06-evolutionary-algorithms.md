---
title: Evolutionary algorithms
published: false
toc: true
---


The goal is to understand evolutionary programming techniques.

## Notes on *Introduction to Evolutionary Computing* 

### Why evolutionary computing


### Problems: what can be solved

Model and desired output are specified (e.g. by a loss function). Task is to find the inputs (which can include model parameters) that lead (as close as possible) to the specified output. 
or 
Input/output pairs are known. Task is to learn a model which delivers the correct output for each known input. 

#### NP Problems
In this domain, problem categories are defined through the properties of problem-solving algorithms. 

If the search space $S$ is defined by continuous variables, then the problem is a **numerical optimization problem**. If $S$ is defined by discrete variables, the problem is a **combinatorial optimization problem**. 

Computational complexity is determined by problem size (dimensionality of the problem at hand; number of variables) and the (worst-case) running time. Superpolynomial (e.g., exponential) run-time is generally unacceptable. 

Problem reduction means that we can transform one problem into another via a suitable mapping. 

A problem belongs to $\mathbf{P}$ if there exists an algorithm that can solve it in polynomial time. $\mathbf{NP}$ contains problems whose solutions can be verified within polynomial time. $\mathbf{P}$ is a subset of $\mathbf{NP}$, as any polynomial-time *solver* can be used as a *verifier*. 

A problem belongs to the class $\mathbf{NP}$-**complete** if it belongs to the class $\mathbf{NP}$ and any other problem in $\mathbf{NP}$ can be reduced to this problem (i.e. NP-complete is the "hardest in NP) by an algorithm which runs in polynomial time. 

A problem belongs to the class $\mathbf{NP}$-**hard** if it is at least as hard as any problem in $\mathbf{NP}$-**complete** (all problems in NP-complete reduce to one in NP-hard), but the solutions cannot necessarily be verified in polynomial time. 

```
Initialize population with random candidate solutions
Evaluate each candidate
UNTIL termination condition is satisfied:
	Select parents;
	Recombine pairs of parents;
	Mutate offspring;
	Evaluate new candidates;
	Select individuals for the next generation;
```


## Notes on [FunSearch](https://www.nature.com/articles/s41586-023-06924-6)

