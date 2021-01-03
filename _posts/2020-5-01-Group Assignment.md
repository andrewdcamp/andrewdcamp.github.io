---
title: "Optimal Grouping"
date: "2020-05-10"
tags: [Machine Learning, R]
header:
  image: "/images/railroad.jpg"
excerpt: "Using clustering and linear programming to assign members to groups based on multiple attributes"
---

## Background
One of my family members is a director for a non-profit Houston area women’s bible study. Each year, one of her tasks is placing hundreds of women into study groups for the year. She does her best to place these women into groups which maximize the difference in the backgrounds of the individuals in the group, to break people out of their comfort zones and foster new perspectives. Each year, the organization’s directors get together and physically write the names of the members and their background attributes – denomination, age group, etc. onto index cards. They spend multiple days physically reorganizing the cards into groups in an iterative manner. The result is equal groups of women with a broad array of backgrounds.

This sounded like a job for technology to me, so I offered to try to put something together to automate the effort.

This is an alternative approach to the solution in part 1 of this project, where I attempted to used linear programming to solve the same problem.

### Here are the rules
* The groups must be as close to 16 people as possible (one over or under is okay) with a single leader assigned to each
+ The ideal assignment would “mix up” the members by various attributes so each group has as little in common as possible
- If members participated the year before, they should ideally have a different group leader this year

### The data
The non-profit has an internal system where sign up data with the list of the upcoming year’s members can be found along with their denomination, age group, and their leader from last year.

The data has been pseudonymized to keep personal info private.


| Name              | Age Group               | Denomination       | Original Leader    |
|:------------------|:------------------------|:-------------------|:-------------------|
| Kissell, Samantha | 1946-1964(Boomers)      | Catholic           | Stevens, Elizabeth |
| Winkler, Selena   | 1965-1980(Generation X) | Catholic           | Olsen, Zoe         |
| Laouar, Jaymin    | 1946-1964(Boomers)      | Non Denominational | Zordel, Linden     |
| Nicolls, Sarah    | 1965-1980(Generation X) | Non Denominational | Speakman, Rebecca  |
| Fort, Kristyn     | 1965-1980(Generation X) | Methodist          | Chapman, Sarah     |


## Problem solving approach
### Assigning groups
Clustering is a great way to organize data into groups, though most clustering algorithms are not designed to deliver same-size clusters. I found an excellent comparison of same size clustering methods from [this post](http://jmonlong.github.io/Hippocamplus/2018/06/09/cluster-same-size/) and after trying out a few approaches, modified the hierarchical bottom clustering approach to fit my needs.

The function accepts a distance matrix as an input and follows an iterative approach to place data into groups given a cluster size.

To create the input matrix, we create *similarity* matrices for each of our members’ attributes, providing a positive score where an attribute overlaps between two members, weighted by arbitrary importance. I.e. if two people are in the same age group we add a value to their similarity score.


|                   |Adcock, Lillian |Albro, Claire |Alexander, Sabryna |Allen, Arielle |
|:------------------|:---------------|:-------------|:------------------|:--------------|
|Adams, Olivia      |0               |0             |0                  |0              |
|Adcock, Lillian    |                |0             |5                  |5              |
|Albro, Claire      |                |              |0                  |0              |
|Alexander, Sabryna |                |              |                   |5              |


The three independent matrices are then added to create a cumulative similarity matrix between all members.


|                   |Adcock, Lillian |Albro, Claire |Alexander, Sabryna |Allen, Arielle |
|:------------------|:---------------|:-------------|:------------------|:--------------|
|Adams, Olivia      |3               |0             |0                  |0              |
|Adcock, Lillian    |                |0             |5                  |5              |
|Albro, Claire      |                |              |0                  |0              |
|Alexander, Sabryna |                |              |                   |15             |


We now have similarity measures for our individuals, but since we are aiming to group our members together based on dissimilarity, and the hierarchical clustering function accepts a distance matrix, we can just reverse our logic and use the similarity scores as *distance* scores. I.e. we want to cluster the data such that the most similar individuals are as far apart as possible.

### Clustering
Approach

We get a result with 400+ members in a matter of a few seconds.

### Taking a look at the output

| Cluster|Name               |Original Leader  |Age Group               |Denomination     |
|-------:|:------------------|:----------------|:-----------------------|:----------------|
|       1|Adams, Olivia      |Monson, Shauna   |1981-2000(Millennials)  |Methodist        |
|      26|Adcock, Lillian    |Odalen, Brittany |1965-1980(Generation X) |Methodist        |
|       1|Albro, Claire      |Mingus, Crystal  |1946-1964(Boomers)      |Baptist          |
|       1|Alexander, Sabryna |Pratt, Emilee    |1965-1980(Generation X) |Lutheran         |
|      32|Allen, Arielle     |Pratt, Emilee    |1965-1980(Generation X) |Church Of Christ |
|      26|Althoff, Holly     |Clark, Taylor    |1946-1964(Boomers)      |Baptist          |


Groups/clusters are even - 16 members in each – testing with different data has yielded a few groups with 15 members due to different member totals - which is acceptable given our goals.


| Cluster| Count|
|-------:|-----:|
|       1|    16|
|       2|    16|
|       3|    16|
|       4|    16|
|       5|    16|
|       6|    16|
|     ...|   ...|
|      36|    16|


Aggregating the input data, we can view the percentage of members in each age group. Then, aggregating our results, we can compare the average AgeGroup count per cluster to the ideal numbers above. It's highly unlikely that the ideal will be reached, but the comparison provides a sanity check that our approach is working. The same trend holds true for each of our attributes.


|Age Group               |  pct| Ideal Per Cluster| Average Per Cluster |
|:-----------------------|----:|-----------------:|--------------------:|
|1900-1945(Builders)     | 0.06|                 1|                 1.19|
|1946-1964(Boomers)      | 0.44|                 7|                 6.97|
|1965-1980(Generation X) | 0.39|                 6|                 6.28|
|1981-2000(Millennials)  | 0.11|                 2|                 1.72|



We now have evenly sized groups that are effectively *mixed up* or clustered by dissimilarity. Next we need to assign an ideal leader to each group.

### Leader Assignment
We want to assign leaders to groups in a way that minimizes the number of people who have the same leader as last year.A list of this year’s leaders is provided from the non-profit’s same data system. If the list contains fewer leaders than groups for some reason, we can add “dummy” leaders (*Leader_1, Leader_2, …*) to our list for a user to replace later, and if there are more leaders on the list than `n` clusters, we will only use the first `n` leaders.

We use a well known operations research approach - the assignment problem - to place our leaders with a group. Cost of assigning a leader to a group is defined by the number of individuals in the group who had that particular leader last year. This cost info is defined in a cost matrix like the one below.


|               |11 |18 |21 |28 |31 |34 |36 |
|:--------------|:--|:--|:--|:--|:--|:--|:--|
|Adler, Shelby  |1  |1  |1  |0  |1  |1  |0  |
|Brown, Hannah  |   |0  |0  |1  |0  |0  |0  |
|Chapman, Sarah |   |   |1  |1  |1  |1  |1  |
|Cheney, Amanda |   |   |   |1  |0  |0  |0  |
|Clark, Taylor  |   |   |   |   |0  |0  |0  |
|Gaito, Sophia  |   |   |   |   |   |1  |1  |
|Garrett, Kyla  |   |   |   |   |   |   |0  |


The `lpsolve` library has a very nice function for solving the assignment problem without manually formulating the model. Here we define a model to minimize the overall cost by assigning one leader to each group.

```r
    lpassign <- lp.assign(LeaderMtrx, direction = "min")
```

Results show that **0** individuals will have the same leader as last year. Excellent!

```r
    > lpassign$objval
    [1] 0
```

### Results
We now have clustered groups with an ideal leader assigned to each. This checks the boxes on our goals!

| Cluster|Name               |Original Leader       |Age Group               |Denomination |Leader            |
|-------:|:------------------|:---------------------|:-----------------------|:------------|:-----------------|
|       1|Adams, Olivia      |Monson, Shauna        |1981-2000(Millennials)  |Methodist    |Williams, Rebecca |
|       1|Doyle, Kortney     |Garrett, Kyla         |1965-1980(Generation X) |Baptist      |Williams, Rebecca |
|       1|Albro, Claire      |Mingus, Crystal       |1946-1964(Boomers)      |Baptist      |Williams, Rebecca |
|       1|Alexander, Sabryna |Pratt, Emilee         |1965-1980(Generation X) |Lutheran     |Williams, Rebecca |
|       1|Jeranko, Shianne   |Chapman, Sarah        |1946-1964(Boomers)      |Baptist      |Williams, Rebecca |
|       1|Strauch, Tyler     |Hristopoulos, Harmony |1946-1964(Boomers)      |Baptist      |Williams, Rebecca |


The solution is put together in an [R Shiny application](https://adcamp.shinyapps.io/group_assignment/) accessible from a web browser. A user can upload relevant files, and download the results.

In comparison to the solution in part 1 of this project, this approach is simple and much more effective.

The overall process for the user is very quick – much better than multiple days rearranging index cards.
