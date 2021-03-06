


# pyschedule overview

pyschedule is a package to formulate and solve resource-constrained scheduling problems, which covers a lot! Its main modelling entities are:

- **precedence relations between tasks** (e.g. this one is before this one)
- **resource requirements of tasks** (e.g. this one can be done either by her or him)
- **capacities** (e.g. he can only do that many)

more specifically, pyschedule ...

- ... is not a general optimization language but only targets the scheduling use case. It tries to be as simple as possible but not simpler
- ... is organized in scenarios where a scenario is a list of tasks and resources
- ... exploits the python language to provide a syntax that is comparable to the usual declarative description languages
- ... scenarios can be solved using different solvers which are not part of pyschedule
- ... also offers [gantt](https://en.wikipedia.org/wiki/Gantt_chart)-chart type of plotting, since optimization without visualization does not work

### Solvers

pyschedule tries to gain the best of two worlds by supporting classical MIP-models (Mixed Integer) as well as CP solvers (Constraint Programming). All solvers interfaces are in the solvers subpackage:

- **mip.solve(scenario,kind) :** time-indexed MIP-formulation build on top of package [pulp](https://github.com/coin-or/pulp). Use parameter "kind" to select the MIP-solver ("CBC" (default if kind is not provided), "CPLEX", "GLPK", "SCIP", "GUROBI"). CBC is part of pulp and hence works out of the box. For CBC, CPLEX, SCIP and GUROBI you can pass parameters "time_limit" or "ratio_gap" (only CBC and SCIP) to limit the running time and receive suboptimal solutions.
- **mip_bigm.solve(scenario,kind) :** classical bigM-type MIP-formulation, works for small models.

There are some more solvers not based on MIPs, but they are not supported that well, so please dont use yet (volunteers welcome!):

- **ortools.solve(scenario) :** the open source CP-solver of Google, a little restricted but good to ensure feasibility of larger models. Make sure that package [ortools](https://github.com/google/or-tools) is installed.
- **cpoptimizer.solve(scenario) :** [IBM CP Optimizer](http://www-01.ibm.com/software/commerce/optimization/cplex-cp-optimizer/), requires command "oplrun" to be executable. Industrial-scale solver that runs fast on very large problems.
- **cpoptimizer.solve_docloud(scenario,api_key) :** IBM CP Optimizer hosted in the cloud, you need to provide an "api_key" which you can get [here](https://developer.ibm.com/docloud/) for a trial.
- to be continued ...

There is one pre-defined heuristic (meta-solver):

- **listsched.solve(scenario,solve_method,task_list,batch_size) :** all tasks are added in the order of task_list and integrated in the current schedule using solve_method. If task_list is not specified, then an ordering according to the precedence constraints is used as default.


pyschedule has been tested with python 2.7 and 3.4. All solvers support a parameter "msg" to switch feedback on/off. Moreover, solvers.pulp.solve, solvers.pulp.solve_discrete and solvers.ortools.solve support a parameter "time_limit" to limit the running time.

### Constraints

#### Basic Test Cases
basic test cases that are not really constraints but still represent different capabilities:
- **ZERO :** tasks of length zero are allowed, e.g. `T1 = S.Task(length=0)`.
- **NONUNIT :** tasks of lenght > 1 are allowed, e.g. `T1 = S.Task(length=3)`.

#### Precedences
- **BOUND :** normal bounds, e.g. `T1 > 3` or `T1 < 5`, T1 starts after period 3 or ends before period 5, respectively.
- **BOUNDTIGHT :** tight bounds, e.g. `T1 >= 3` or `T1 <= 5`, T1 starts at period 3 or ends at period 5, respectively.
- **LAX :** lax precedences, e.g. `T1 < T2`, T1 is finished before T2 starts.
- **LAXPLUS :** lax precedences with offset, e.g. `T1 + 3 < T2`, T1 is finished 3 time units before T2 starts. This can be also written as `T2 - T1 > 3`. Note that `T2 - T1 < 3` will ensure that the T2 start at most 3 time units after T1 ends.
- **TIGHT :** tight precedences, e.g. `T1 <= T2`, T2 starts exactly when T1 finishes.
- **TIGHTPLUS :** tight precedences with offset, e.g. `T1 + 3 <= T2`, T2 starts exactly 3 time units after T1 finishes. This can be also written as `T2 - T1 >= 3`. Note that `T2 - T1 <= 3` will ensure that the T2 start exactly 3 time units after T1 ends.
- **COND :** conditional precedences, e.g. `T1 + 3 << T2`, if T1 and T2, then T1 must be finished 3 time units before T2 starts.


#### Resources
each task needs at least one resource. To keep the syntax concise, pyschedule used % as an operator to connect tasks and resources
- **MULT :** multiple resources, e.g. `T1 += [R1,R2]`, T1 uses R1 and R2.
- **ALT :** alternative resources, e.g. `T1 += R1|R2`, T1 uses either R1 or R2. For a list of resources L it is also possible to write `T1 += pyschedule.alt(L)` or `T1 += pyschedule.alt( R for R in L )`.
- **TASKSREQ :** synchronize resources between tasks, e.g. `T1 += T2`, if T2 uses some resource and T1 could use the same resource, then T1 will use this resource. This is useful if tasks T1 and T2 are related, and hence they should use the same resources to avoid switching
- **CUMUL :** cumulative resources, e.g. `R1 = S.Resource('R1',size=3); T1 += R1*2`, R1 has size 3 and T1 uses 2 units of that size whenever run. This can be used to model worker pools or any other resource with a high multiplicity. Using a resource of size n instead n resources of size 1 often helps the solver.
- **CAP :** capacities, e.g. `R1['length'] <= 4`, the sum of the lengths of the tasks assigned to R1 must be at most 4. We can also generate other parameters, e.g. first set `T1.work = 3` and then `R1['work'] <= 4`.
- **CAPSLICE :** capacities, e.g. `R1['length'][:10] <= 4`, the sum of the lengths of the tasks assigned to R1 during periods 0 to 9 must be at most 4. In case a task starts before period 9 and ends after period 9, the capacity requirement of this task is proportional to the overlap
- **CAPDIFF :** change in capacity over time like a derivate, e.g. `R1['length'].diff <= 4`, the number of times resource R switches from running to not running or vice versa is at most 4. We can also use other parameters than length, e.g. first set `T1.work = 3` and then `R1['work'].diff <= 4`.
- **CAPDIFFSLICE :** change of capacity over time in slice, e.g. `R1['length'][:10].diff <= 4`, the number of times resource R switches from running to not running or vice versa in periods 0 to 9 is at most 4.
- **CAPMAX :** maximum capacity at any time, e.g. `R1['length'][:10].max <= 2`, only tasks with length at most 2 are allowed in periods 0 to 9.
- **SCHEDULECOST :** if schedule_cost is set, e.g. `T1.schedule_cost = 1`, then this task is optional, but the schedule_cost will be added from the objective. This is for situations where not all tasks can be scheduled, and hence the most valuable ones with the lowest schedule_cost should be selected. Note that the schedule_cost can also be negative.
- **PERIODS :** restrict the periods of a resource or a task by setting the periods parameter, e.g. `R.periods = [1,2,3]` or `T.periods = [2,5,6]`. This can also be combined


### Solvers vs Constraints
output of [test script](https://github.com/timnon/pyschedule/blob/master/examples/test-solvers.py):

<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>mip.solve</th>
      <th>mip_bigm.solve</th>
    </tr>
    <tr>
      <th>scenario</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>ZERO</th>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>NONUNIT</th>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>BOUND</th>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>BOUNDTIGHT</th>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>LAX</th>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>LAXPLUS</th>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>TIGHT</th>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>TIGHTPLUS</th>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>COND</th>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>ALT</th>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>MULT</th>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>TASKSREQ</th>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>CUMUL</th>
      <td>X</td>
      <td></td>
    </tr>
    <tr>
      <th>CAP</th>
      <td>X</td>
      <td>X</td>
    </tr>
    <tr>
      <th>CAPSLICE</th>
      <td>X</td>
      <td></td>
    </tr>
    <tr>
      <th>CAPDIFF</th>
      <td>X</td>
      <td></td>
    </tr>
    <tr>
      <th>CAPDIFFSLICE</th>
      <td>X</td>
      <td></td>
    </tr>
	<tr>
      <th>CAPMAP</th>
      <td>X</td>
      <td></td>
    </tr>
    <tr>
      <th>SCHEDULECOST</th>
      <td>X</td>
      <td></td>
    </tr>
	<tr>
      <th>PERIODS</th>
      <td>X</td>
      <td></td>
    </tr>
  </tbody>
</table>

X means that the constraint is working and Error means that it produces an error when using it (to be fixed).
