---
title: "Group Assignment - Part 1"
date: "2020-05-10"
tags: [Linear Programming, Julia]
header:
  image: "/images/railroad.jpg"
excerpt: "Assigning members to groups based on multiple attributes using linear programming"
---

## Background
One of my family members is a director for a non-profit Houston area church organization. Each year, one of her tasks is placing hundreds of women into community groups for the year. She does her best to place these women into groups which maximize the difference in the backgrounds of the individuals in the group, to break people out of their comfort zones and foster new perspectives. Each year, the organization’s directors get together and physically write the names of the members and their background attributes – church affiliation, age group, etc. onto index cards. They spend multiple days physically reorganizing the cards into groups in an iterative manner. The result is equal groups of women with a broad array of backgrounds.

This sounded like a job for technology to me, so I offered to put something together to automate the effort.

### The Problem
* The groups must be as close to 16 people as possible (one over or under is okay) with a single leader assigned to each
+ The ideal grouping would “mix up” the members by various attributes so each group has as little in common as possible
- If members participated the year before, they should ideally have a different group leader this year

## Problem solving approach
### Maximizing diversity
There are several ways we could approach measuring difference within each group. Instead of maximizing the differences within each group, we can alternatively choose to minimize the difference in composition between each group and a calulcated "ideal" number for each group. I.e. when all groups are relatively similar in composition, our individuals will be effectively "mixed up".

### Formulating an approach
Each individual will be assigned a one hot coded vector to represent their respective attributes.
To measure a group's composition, each group will have a statistic representing the sum of vectors of individuals assigned to the group.

### The data
The non-profit has an internal system where sign up data with the list of the upcoming year’s members can be found along with their denomination, age group, and their leader from last year.

The data has been pseudonymized to keep personal info private.

| Name              | Age Group               | Affiliation        | Original Leader    |
|:------------------|:------------------------|:-------------------|:-------------------|
| Kissell, Samantha | 1946-1964(Boomers)      | Catholic           | Stevens, Elizabeth |
| Winkler, Selena   | 1965-1980(Generation X) | Catholic           | Olsen, Zoe         |
| Laouar, Jaymin    | 1946-1964(Boomers)      | Non Denominational | Zordel, Linden     |
| Nicolls, Sarah    | 1965-1980(Generation X) | Non Denominational | Speakman, Rebecca  |
| Fort, Kristyn     | 1965-1980(Generation X) | Methodist          | Chapman, Sarah     |

For input data prep and analysis of the model results, I've chosen to use Python. Julia is more than capable of handling this, but personally I can get the job done quicker with Python.

We can one hot encode each individuals attributes to construct a vector describing each of them.

```python
    import pandas as pd

    # read in data
    df = pd.read_csv('/Users/andrewcamp/Documents/Python/CoreGroupOpt/TestData.csv')

    # one hot encode attribute columns
    df_oneHot =(pd.concat(
        [pd.get_dummies(df['AgeGroup']),
        pd.get_dummies(df['Denomination']),
        pd.get_dummies(df['CoreGroupLeader']),
        pd.get_dummies(df['AgeGroup'])], axis = 1))

    # add names back into dataframe
    df_oneHot['Name'] = df['Name']

    # write to folder for input into julia
    df_oneHot.to_csv("/Users/andrewcamp/Documents/Python/CoreGroupOpt/df_optInput.csv", index = False)

```

The task of our model will be to choose which individuals are assigned to each group in order to minimize the difference in composition between all groups. We can achieve this by minimizing the sum of the L1 norm (Manhattan or Taxicab distance) between each group statistic and a calculated vector of ideal attribute counts. Of course we still need to obey the group size constraints.

A sketch of this representation:

In more concrete terms, we can formulate the problem as follows:


### Linearization
Manhattan distance takes the sum of absolute values of distances between vectors. Absolute value is non-linear and therefore we cannot directly implement our objective function via linear programming. To get around this issue, we need to linearize our objective using the approach below:

### Slack variables
We want our groups to be as close to our specified group size as possible, but we will inevitably have a few extra individuals left over. Rather than forcing an exact group size constraint, we add the slack variables s1 and s2 to allow the model to place extra individuals somewhere, but we add our slack variables to the objective with coefficients of 1000 to make sure the model has a strong preference for equal sized groups.

### Model implementation
To solve the problem we can formulate a linear optimization model. A modeling language and external solver are needed to build and solve the model. There are numerous tools in various languages for LP modeling. My favorite is Julia's JuMP package, which I've used here to formulate the LP.

```julia
    using JuMP, CSV, DataFrames, Cbc

    # read one hot coded data
    f = "/Users/andrewcamp/Documents/Python/CoreGroupOpt/df_optInput.csv"
    df = CSV.read(f, DataFrame)
    nms = df.Name
    attrMx = Matrix(select(df, Not(:Name)))

    # input parameters
    Group_size = 5
    N_People = floor(Int,size(df, 1))
    N_Groups = floor(Int,N_People/Group_size)
    AttrVecSize = floor(Int, size(df, 2) - 1)

    # create Model
    #m = Model(solver=GurobiSolver())
    m = Model(Cbc.Optimizer)
    #m = Model(Ipopt.Optimizer)

    # create vars
    @variable(m, y[1:N_Groups,1:N_People], Bin)
    @variable(m, g[1:N_Groups,1:AttrVecSize])
    @variable(m, dist[1:N_Groups, 1:N_Groups, 1:AttrVecSize] >= 0)
    @variable(m, s1 >=0)
    @variable(m, s2 >=0)

    # group stat defintions
    for yRow in 1:size(y, 1)
        for mxCol in 1:size(attrMx, 2)
            @constraint(m, g[yRow, mxCol] == sum(y[yRow,:].*attrMx[:,mxCol]))
        end
    end

    # distance metric defintion
    for i in 1:AttrVecSize
        for j in 1:N_Groups
            for k in 1:N_Groups
                if j < k
                    @constraint(m, g[j, i] - g[k, i] <= dist[j, k, i])
                    @constraint(m, g[k, i] - g[j, i] <= dist[k, j, i])
                end
            end
        end
    end

    # group size with slack vars
    for i in 1:N_Groups
       @constraint(m, sum(y[:,i]) <= (Group_size+s1)) # people assigned to each group < group size
       @constraint(m, sum(y[:,i]) >= (Group_size-s2))
    end

    # each person only one group constraints
    #for j in N_Groups
    for i in 1:N_People
        @constraint(m, sum(y[:,i]) == 1)
    end

    # objective

    #@objective(m, Min, sum(((g[j, i] - g[k, i])^2) for i=1:AttrVecSize,j=1:N_Groups, k=1:N_Groups) + s1 + s2)
    @objective(m, Min, sum(dist[i, j, k] for i=1:N_Groups,j=1:N_Groups, k=1:AttrVecSize) + 5*s1 + 5*s2)

```

To solve our model, we can choose any one of the numerous available commercial or open source solvers supported by JuMP. My top choice for an open source solver is the COIN OR CBC solver.

### Solving the model
Trying out a test case, we build the model to group 160 individuals into 10 groups of 16.
Solving the model we get a solution in about one second.

However with 300 people, our model takes a very long time to solve and if we add our third attribute - original leader into our attribute vectors, the model does not even solve with 160 people...

### Model output
The Good News:
With a test case of 160 individuals we get a solution from CBC rather quickly. Plots of the attribute count in each group are shown below. Notice that due to overlapping attributes (2 per person) the attribute counts are not perfectly even, but we can see that the count is relatively uniform across all groups. We can also see that our groups are even with 16 members in each. Great! - this is what we want.

| Group| Count|
|-------:|-----:|
|       1|    16|
|       2|    16|
|       3|    16|
|     ...|   ...|
|      10|    16|

image:
<img src="{{ site.url }}{{ site.baseurl }}/images/Project1/160_agegroup_plot.png" alt="Plot 1">

The Bad News:
When we try to group 300+ individuals, the solver really struggles to solve the model. Even with a few reformulations of the model I could not get a solution in a reasonable amount of time. With heuristics or more advanced optimization methods, it may be possible to solve this problem, but I ultimately decided to scrap the LP and start over with a clustering approach in R which you can read about in Part II of this post.

###Concluding thoughts
Though I was a bit disappointed not to solve this problem on the first try, though even in academic literature this type of grouping problem is nowhere near trivial. This was a great exercise in implementing a mathematical formulation to solve a real problem and I've become a big fan of using JuMP optimization modeling.

Check out Part II of this post to read about the alternative solution using clustering in R. (Spoiler - the second approach was successful)
