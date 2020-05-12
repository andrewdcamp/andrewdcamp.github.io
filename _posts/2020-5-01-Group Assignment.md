---
title: "Group Assignment"
date: "2020-05-10"
tags: [Machine Learning, R]
header:
  image: "/images/railroad.jpg"
excerpt: "Using clustering and linear programming to assign members to groups"
---

## Background
One of my family members is a director for a non-profit Houston area women’s bible study. Each year, one of her tasks is placing hundreds of women into study groups for the year. She does her best to place these women into groups which maximize the difference in the backgrounds of the individuals in the group, to break people out of their comfort zones and foster new perspectives. Each year, the organization’s directors get together and physically write the names of the members and their background attributes – denomination, age group, etc. onto index cards. They spend multiple days physically reorganizing the cards into groups in an iterative manner. The result is equal groups of women with a broad array of backgrounds.

This sounded like a job for technology to me, so I offered to try to put something together to automate the effort.

### Here are the rules:
* The groups must be as close to 16 people as possible (one over or under is okay) with a single leader assigned to each
+ The ideal assignment would “mix up” the members by various attributes so each group has as little in common as possible
- If members participated the year before, they should ideally have a different group leader this year

### The data:
The non-profit has an internal system where sign up data with the list of the upcoming year’s members can be found along with their denomination, age group, and their leader from last year.

The data has been pseudonymized to keep personal info private.

| Name              | AgeGroup                | Denomination       | CoreGroupLeader    |
|:------------------|:------------------------|:-------------------|:-------------------|
| Kissell, Samantha | 1946-1964(Boomers)      | Catholic           | Stevens, Elizabeth |
| Winkler, Selena   | 1965-1980(Generation X) | Catholic           | Olsen, Zoe         |
| Laouar, Jaymin    | 1946-1964(Boomers)      | Non Denominational | Zordel, Linden     |
| Nicolls, Sarah    | 1965-1980(Generation X) | Non Denominational | Speakman, Rebecca  |
| Fort, Kristyn     | 1965-1980(Generation X) | Methodist          | Chapman, Sarah     |

## Problem solving approach:
### Assign groups:
Clustering is a great way to organize data into groups, though most clustering algorithms are not designed to deliver same-size clusters. I found an excellent comparison of same size clustering methods from [this post](http://jmonlong.github.io/Hippocamplus/2018/06/09/cluster-same-size/) and after trying out a few approaches, modified the hierarchical bottom clustering approach to fit my needs.

The function accepts a distance matrix as an input and follows an iterative approach to place data into groups given a cluster size.

To create the input matrix, we create *similarity* matrices for each of our members’ attributes, providing a positive score where an attribute overlaps between two members, weighted by arbitrary importance with a bit of trial and error.

The three independent matrices are then added to create a cumulative similarity matrix between all members.

Since we are aiming to group our members together based on dissimilarity, and the hierarchical clustering function accepts a distance matrix, we can invert our logic and use the similarity scores as distance scores to cluster “backwards”.

###Taking a look at the clustered output:
Groups/clusters are mostly even. 16 members in each – a few groups with 15 members, which is acceptable given our goals.

The average and standard deviation of our attributes by group show that members are evenly “mixed up” by their attributes.

We now have groups that meet our criteria very effectively. Now we need to assign an ideal leader to each group.

### Leader Assignment
A list of this year’s leaders is provided from the non-profit’s same data system. If we have fewer leaders than groups, we can add “fake” leaders (Leader_1, Leader_2, …) to our list to replace later, and if there are more leaders than n groups, we will only use the first `n` leaders for `n` clusters.

We use a linear programming approach - the assignment problem - to place our leaders with a group. Cost of assigning a leader to a group (cluster) is defined by the number of individuals in the group who had a particular leader last year. This cost info is defined in a cost matrix like the one below.
Fake leaders are scaled by 10x to avoid precedence issues with actual leaders.
Using the `lpsolve` library we formulate a model to minimize the overall cost by assigning one leader to each group.

Results show that 0 individuals will have the same leader as last year. Excellent! This checks the box on each of our criteria.
```r
    lpassign$objval
    [1] 0
```
The solution is put together in an [R Shiny application](https://adcamp.shinyapps.io/group_assignment/) accessible from a web browser. A user can upload relevant files, and download the results.

The overall process is very quick – much better than multiple days rearranging index cards.
