---
layout: single
permalink: /nonlinearsdpcuts
excerpt: "The first cut is not the deepest"
title: "SDP cones in BMINBB"
tags: [Global optimization, Semidefinite programming]
comments: true
published: false
date: 2020-10-02
---

YALMIP has an internal very general global solver [BMIBNB](/solver/bmibnb). Since inception some 15 years ago, it has supported nonlinear semidefinite constraints (this is actually why it initially was developed, hence the name) but this feature has essentially been useless as it has required a nonlinear SDP solver for the upper bound and solution generation. Although [PENSDP](/solver/pensdp) and [PENBMI](/solver/penbmi) are such solvers, they are not robust enough to be used in the branch & bound framework, and the development of these solvers appear to have stalled.

With the most recent release, the situation is improved, and there is now better support for the semidefinite cone in [BMIBNB](/solver/bmibnb), and a completely different approach is used to achieve this. Instead of relying on external (non-existant) nonlinear SDP solvers, the machinery for the upper bound problem is based on a nonlinear cutting plane strategy using your standard favorite nonlinear solver. Will this work on any problem? Absolutely not. You might be lucky though, and this is work in progress.

## Suitable background

To avoid repetition of general concepts, you are adviced to read the post on [CUTSDP](/cutsdpsolver) where cutting planes are used for the linear semidefinite cone in a different context. You are also adviced to read the general posts on [global optimization] and [envelope generation](tutorial/envelopesinbmibnb/) to understand how a spatial branch & bound solver works.

## The strategy

Now, having read the recommended articles, you know that [BMIBNB](/solver/bmibnb) needs two central components: A lower bound solver which solves a convex typically linear relaxation of the problem, and an upper bound solver which tries to solve the original nonlinear problem starting from some suitably selected starting point (typically some combination of the center of the currently investigated box and the solution to the lower bound problem)

The lower bound problem does not pose any particular problem when we add a semidefinite cone to the model. After relaxing nonlinearities and all we have to do is to solve a linear semidefinte program, which we have a plethora of solvers for. Hence, the only difference is that we swtich from a linear programming based lower bound (or more generally a quadratic or second-order cone programming solver) to a linear semidefinite programming solver.

The upper bound is the issue. While there are many solvers available for standard nonlinear programming, this is not the case for nonlinear semidefinite programming. In principle though, if our nonlinear model has a semidefinite constraint \\(G(x) \succeq 0\\), possibly nonlinear, we can view this as \\(v^TG(x)v \geq 0~\forall v\\). Hence, by simply adding an infinite amount of scalar constraints, we can mimic the semidefinite constraint and solve the upper bound problem using a standard nonlinear solver. Of course, this is not practical or possible. Instead, we use the idea iteratively and build an approximation of the semidefinite cone as we work our way through the tree. Remember, it is not crucial that we actually manage to compute an upper bound in every node. It is of course good if we can generate feasible solutions, but if we fail, it only means we have to open more nodes and try to find feasible solutions with associated upper bounds in those future nodes. Also remember that the upper bound problem is solved on a globally valid model. A semidefinite cut added to the upper bound model is valid in all nodes.

Hence, the basic algorithm is

1. Create some initial approximation \\(\hat{G}(x) \geq 0\\) of the semidefinite cone (e.g. add the constraint that the diagonal is non-negative)

2. In the node, try to solve the nonlinear program using the approximation \\(\hat{G}(x)\\). If a solution is found, and \\(G(x) \succeq 0\\) is satisfied, a valid upper bound has been computed. If \\(G(x) \succeq 0\\) is violated, compute an eigenvector \\(v\\) associated to a negative eigenvalue and add the cut \\( v^TG(x)v \geq 0 \\) to the approximation \\(\hat{G}(x)\geq 0\\).

3. Either repeat (2) several times in the node, or proceed with branching process and use the new approximation first in the next node.

Everything else is exactly as usual in the global algorithm.

## Illustrating the steps manually

Consider a small example where we want to minimize \\(x^2+y^2\\) over \\(-3 \leq (x,y) \leq 3\\) and \\( G(x,y) = \begin{pmatrix} y^2-1 x+y\\x+y 1\end{pmatrix} \succeq 0\\). The semidefinite constraints can be written as \\(y^2-1\geq 0\\) and \\( y^2-1 - (x+y)^2 \geq 0\\) by studying conditions for the matrix to be positive semidefinite (the first condition is redundant as it is implied by the second)

The feasible set are the two disjoint regions top left and bottom right with borders outlined in red in the figure below

![True feasible set]({{ site.url }}/images/nonlinearsdpcut1.png){: .center-image }

THe figure can be generated with the following code

````matlab
clf;
l=ezplot('y^2-1-(x+y)^2',[-3 3 -3 3]);
set(l,'color','red');set(l,'linewidth',2);
hold on
grid on
````

As a starting approximation of the feasible set, in addition to the box constrants, we have the diagonal requirement \\(y^2 -1 \geq 0\\). Obviously, this is the region above the top black line, and below the bottom black line in the following figure

![Approximation 1]({{ site.url }}/images/nonlinearsdpcut2.png){: .center-image }

To generate these lines we ran

````matlab
l=ezplot('y^2-1-0*x',[-3 3 -3 3]);
set(l,'color','black');set(l,'linewidth',2);
````

Now assume we use this initial model for upper bound generation. A nonlinear solver might find the solution \\(x=0, y=1\\) marked with a black star in the figure below. Plugging in this solution into \\(G(x,y)\\) and computing eigenvalues shows that the matrix is indefinite (it is obviously not positive definite since the solution is outside the feasible region) meaning we failed to find a feasible solution to the original problem and thus generated no upper bound. By computing the associated eigenvector and forming the cut, we obtain a feasible region whose border is shown in blue. The feasible region to this cut is the area to the left of the blue curve. Note that it is tangent to the true feasible set at two points.Our nonlinear model is now the intersection of the two regions defined by the black lines, and the new region *to the left of the blue line*.

![Approximation 2]({{ site.url }}/images/nonlinearsdpcut3.png){: .center-image }

The computations are straightforward

````matlab
l = plot(0,1,'ko');
set(l,'MarkerFaceColor','black')
sdpvar x y
G = [y^2-1 x+y;x+y 1]
assign([x;y],[0;1])
[V,D] = eig(value(G));
gi = V(:,1)'*G*V(:,1);
s = sdisplay(gi);
l = ezplot(s{1},[-3 3 -3 3]);
set(l,'color','blue');set(l,'linewidth',2);
````

Once again, either in the same node or later on, the new model is used by the nonlinear solver, and this time we might obtain \\(x=0, y=-1\\) marked with a blue star. We are still infeasible in the original model and analyzing eigenvalues and generating a new cut leads to the region *to the right of the green curve*. Iterating again using the intersection of the three constraints leads to, say, (-0.7,1) and a cut ilustrated by *the yellow line which we should be to the left of*.

![Approximation 2]({{ site.url }}/images/nonlinearsdpcut5.png){: .center-image }

````matlab
assign([x;y],[0;-1])
l = plot(0,-1,'bo');
set(l,'MarkerFaceColor','blue')
[V,D] = eig(value(G));
gi = V(:,1)'*G*V(:,1);
s = sdisplay(gi);
l = ezplot(s{1},[-3 3 -3 3]);
set(l,'color','green')
set(l,'linewidth',2);
l = plot(-.7,1,'go');
set(l,'Markersize',5)
set(l,'MarkerFaceColor','green')
assign([x;y],[-.7;1])
[V,D] = eig(value(G));
gi = V(:,1)'*G*V(:,1);
s = sdisplay(gi);
l = ezplot(s{1},[-3 3 -3 3]);
set(l,'color','yellow');
set(l,'linewidth',2);
````

## Running the solver

So let us throw [MIBNB](/solver/bmibnb) on the problem (we save some internal history for later analysis). From the log, we can deduce that the algorithm found a feasible solution already in the first node (as we will see later, this is a trivial point in the top left corner). It then saw a bunch of SDP-infeasible solutions from which it generated cuts to add to the model, and after 5 iterations it found a new better solution. More cuts wre added, and in iteration 9 and 15 better solutions were found, and with the solution found in node 15 the gap could be closed completely and thus a globally optimal solution was found.Note that the logtells us that the solutions actually weren't found by the upper bound solver [FMINCON](/solver/fmincon), but were recovered using heuristics from lower bound solutions etc.

````matlab
sdpvar x y
G = [y^2-1 x+y;x+y 1]
Model = [-3 <= [x y] <= 3, G >= 0];
sol = optimize(Model,x^2+y^2,sdpsettings('solver','bmibnb','savesolveroutput',1));
* Starting YALMIP global branch & bound.
* Upper solver     : fmincon
* Lower solver     : MOSEK
* LP solver        : GUROBI
* -Extracting bounds from model
* -Perfoming root-node bound propagation
* -Calling upper solver (no solution found)
* -Branch-variables : 2
* -More root-node bound-propagation
* -Performing LP-based bound-propagation 
* -And some more root-node bound-propagation
* Starting the b&b process
 Node       Upper       Gap(%)       Lower     Open   Time
    1 :   1.80000E+01   161.90    1.00000E+00    2     0s  Solution found by heuristics | Added 1 cut on SDP cone  
    2 :   1.80000E+01   161.90    1.00000E+00    3     0s  Added 1 cut on SDP cone  
    3 :   1.80000E+01   152.45    1.56312E+00    4     0s  Added 1 cut on SDP cone  
    4 :   1.80000E+01   152.45    1.56312E+00    5     0s  Added 1 cut on SDP cone  
    5 :   2.46872E+00    30.03    1.56312E+00    6     0s  Improved solution found by heuristics | Added 1 cut on SDP cone  
    6 :   2.46872E+00    30.03    1.56312E+00    5     0s  Added 1 cut on SDP cone | Pruned stack based on new upper bound  
    7 :   2.46872E+00    28.68    1.59858E+00    4     0s  Added 1 cut on SDP cone | Pruned stack based on new upper bound  
    8 :   2.46872E+00    28.68    1.59858E+00    5     0s  Added 1 cut on SDP cone  
    9 :   1.68243E+00     3.18    1.59858E+00    6     0s  Improved solution found by heuristics | Added 1 cut on SDP cone  
   10 :   1.68243E+00     3.18    1.59858E+00    3     0s  Poor bound in lower, killing node | Pruned stack based on new upper bound  
   11 :   1.68243E+00     2.60    1.61371E+00    4     0s  Added 1 cut on SDP cone  
   12 :   1.68243E+00     2.60    1.61371E+00    5     0s  Added 1 cut on SDP cone  
   13 :   1.68243E+00     2.60    1.61371E+00    6     0s  Added 1 cut on SDP cone  
   14 :   1.68243E+00     2.60    1.61371E+00    7     0s  Added 1 cut on SDP cone  
   15 :   1.61808E+00     0.00    1.61803E+00    0     0s  Poor bound in lower, killing node | Pruned stack based on new upper bound  
* Finished.  Cost: 1.6181 (lower bound: 1.618, relative gap 0%)
* Termination with all nodes pruned 
* Timing: 39% spent in upper solver (14 problems solved)
*         8% spent in lower solver (19 problems solved)
*         32% spent in LP-based domain reduction (76 problems solved)
*         3% spent in upper heuristics (73 candidates tried)
````

## There's more to the story

As explained above, nothing prevents us from iterating the cut-generating strategy in every node several times. This can e controlled by an option. If we want to do, say, three rounds of solving the nonlinear program followed by adding violated SDP cuts, we run the code below. We see that several rounds are performed in every node, and the result is that more effort is spent in every node but a fewer number of nodes are needed. Note that it is not necessarily a good thing to add many new cuts in every node and strive for an SDP feasible solution everywhere. We might be wasting a lot of resources in creating a high-fidelity model in a region which is far from the globlly optimal solution anyway.

````matlab
sdpvar x y
G = [y^2-1 x+y;x+y 1]
Model = [-3 <= [x y] <= 3, G >= 0];
sol = optimize(Model,x^2+y^2,sdpsettings('solver','bmibnb','bmibnbuppersdprelax',3));
* Starting YALMIP global branch & bound.
* Upper solver     : fmincon
* Lower solver     : MOSEK
* LP solver        : GUROBI
* -Extracting bounds from model
* -Perfoming root-node bound propagation
* -Calling upper solver (no solution found)
* -Branch-variables : 2
* -More root-node bound-propagation
* -Performing LP-based bound-propagation 
* -And some more root-node bound-propagation
* Starting the b&b process
 Node       Upper       Gap(%)       Lower     Open   Time
    1 :   1.80000E+01   161.90    1.00000E+00    2     0s  Solution found by heuristics | Added 1 cut on SDP cone | Added 1 cut on SDP cone | Added 1 cut on SDP cone  
    2 :   1.80000E+01   161.90    1.00000E+00    3     0s  Added 1 cut on SDP cone | Added 1 cut on SDP cone | Added 1 cut on SDP cone  
    3 :   1.80000E+01   152.45    1.56312E+00    4     0s  Added 1 cut on SDP cone | Added 1 cut on SDP cone | Added 1 cut on SDP cone  
    4 :   1.80000E+01   152.45    1.56312E+00    5     0s  Added 1 cut on SDP cone | Added 1 cut on SDP cone | Added 1 cut on SDP cone  
    5 :   2.46872E+00    30.03    1.56312E+00    6     0s  Improved solution found by heuristics | Added 1 cut on SDP cone | Added 1 cut on SDP cone | Improved solution found by upper solver  
    6 :   2.46870E+00    30.03    1.56312E+00    5     0s  Added 1 cut on SDP cone | Added 1 cut on SDP cone | Improved solution found by upper solver | Pruned stack based on new upper bound  
    7 :   2.46870E+00    28.68    1.59858E+00    4     0s  Added 1 cut on SDP cone | Added 1 cut on SDP cone | Added 1 cut on SDP cone | Pruned stack based on new upper bound  
    8 :   1.61803E+00     0.75    1.59858E+00    5     0s  Added 1 cut on SDP cone | Improved solution found by upper solver  
* Finished.  Cost: 1.618 (lower bound: 1.5986, relative gap 0.74589%)
* Termination with relative gap satisfied 
* Timing: 69% spent in upper solver (9 problems solved)
*         5% spent in lower solver (12 problems solved)
*         14% spent in LP-based domain reduction (38 problems solved)
*         1% spent in upper heuristics (36 candidates tried)
````

This can be taken to the extreme. By setting the number of rounds to inf, we force [BMIBNB](solver/bmibnb) to continue adding SDP cuts until the solution either is SDP-feasible, or the nonlinear solver fails to find a solution to the approximation.

Another use of the framework is to use it in combintion with a global solver. By using a global solver for the upper bound solver, and then setting the number of rounds to inf, the solution computed will not only be SDP-feasible, but globally optimal! Since we know any computed solution is the globally optimal solution, we want to terminate without bothering about the lower bound, and we can achive this by setting the lower bound target. Since our SDP cuts are quadratic, and the objective is quadratic, we can use [Gurobi](/solver/gurobi) which can do global optimization for quadratic models. THere is one tricky caveat here though: The numerical tolerances in vs judging feasibility can infer with each other. A solution which [Gurobi](/solver/gurobi) thinks is good enough might still violate the cut we just added, and then we will keep adding the same cut. To counteract this, we tighten the feasibility tolerance in [Gurobi](/solver/gurobi).

````matlab
sdpvar x y
G = [y^2-1 x+y;x+y 1]
Model = [-3 <= [x y] <= 3, G >= 0];
ops = sdpsettings('solver','bmibnb','bmibnb.uppersdprelax',inf,'bmibnb.lowertarget',-inf);
ops = sdpsettings(ops,'bmibnb.uppersolver','gurobi','gurobi.FeasibilityTol',1e-8);
sol = optimize(Model,x^2+y^2,ops);
* Starting YALMIP global branch & bound.
* Upper solver     : GUROBI
* Lower solver     : MOSEK
* LP solver        : GUROBI
* -Extracting bounds from model
* -Perfoming root-node bound propagation
* -Calling upper solver (found a solution!)
* -Branch-variables : 2
* -More root-node bound-propagation
* -Performing LP-based bound-propagation 
* -And some more root-node bound-propagation
* -Terminating in root as solution found and lower target is -inf
* Timing: 79% spent in upper solver (1 problems solved)
*         7% spent in lower solver (4 problems solved)
*         0% spent in LP-based domain reduction (0 problems solved)
*         0% spent in upper heuristics (0 candidates tried)
````

Yet another use is to use it as a trick to create a local solver.Just run it with any nonlinear solver until SDP feasible, and then terminate by setting lower bound target to negative infinity. Note in the display below that in the root node, [fmincon](/solver/fmincon) fails to even find a feasile solution to the SDP relaxed problem, and then no cuts can be generated. THe SDP-feasible solution is actually found by heuristics for thi particular eample.

````matlab
sdpvar x y
G = [y^2-1 x+y;x+y 1]
Model = [-3 <= [x y] <= 3, G >= 0];
ops = sdpsettings('solver','bmibnb','bmibnb.uppersdprelax',inf,'bmibnb.lowertarget',-inf);
ops = sdpsettings(ops,'bmibnb.uppersolver','fmincon');
sol = optimize(Model,x^2+y^2,ops);
* Starting YALMIP global branch & bound.
* Upper solver     : fmincon
* Lower solver     : MOSEK
* LP solver        : GUROBI
* -Extracting bounds from model
* -Perfoming root-node bound propagation
* -Calling upper solver (no solution found)
* -Branch-variables : 2
* -More root-node bound-propagation
* -Performing LP-based bound-propagation 
* -And some more root-node bound-propagation
* Starting the b&b process
 Node       Upper       Gap(%)       Lower     Open   Time
    1 :   1.80000E+01   161.90    1.00000E+00    2     0s  Solution found by heuristics  
* Finished.  Cost: 18 (lower bound: 1, relative gap 161.9048%)
* Termination with upper bound limit reached 
* Timing: 72% spent in upper solver (1 problems solved)
*         5% spent in lower solver (5 problems solved)
*         4% spent in LP-based domain reduction (4 problems solved)
*         2% spent in upper heuristics (4 candidates tried)
````
