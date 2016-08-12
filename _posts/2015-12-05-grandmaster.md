---
layout: post
title: Boxing's Grandmasters
description: "Analysis of Historical boxing data"
modified: 2016-07-31
tags: [boxing,data,science,elo]
comments: true
published: true
image:
  feature: 
  credit: 
  creditlink: 
---

<section id="table-of-contents" class="toc">
  <header>
    <h3>Contents</h3>
  </header>
<div id="drawer" markdown="1">
*  Auto generated table of contents
{:toc}
</div>
</section><!-- /#table-of-contents -->


As a lifelong boxing fan and someone whose everyday job involves analyzing large amounts of data, I thought it would be interesting to apply some quantitative analysis to a sport which has many subjective aspects to it.

## ELO rating system

The Elo rating system is a method for calculating the relative skill levels of players in competitor-versus-competitor games.   The difference in the ratings between two players serves as a predictor of the outcome of a match. Two players with equal ratings who play against each other are expected to score an equal number of wins. 
<br>
<br>
If Player A has a rating of $${\displaystyle R_{A}}$$ and Player B a rating of $${\displaystyle R_{B}}$$, the exact formula (using the logistic curve) for the expected score of Player A is

<center>
$${\displaystyle E_{A}={\frac {1}{1+10^{(R_{B}-R_{A})/400}}}}$$
</center>

Similarly the expected score for Player B is
<center>
$${\displaystyle E_{B}={\frac {1}{1+10^{(R_{A}-R_{B})/400}}}}$$
</center>

This could also be expressed by
<center>
$${\displaystyle E_{A}={\frac {Q_{A}}{Q_{A}+Q_{B}}}} $$
</center>
and

<center>
$${\displaystyle E_{B}={\frac {Q_{B}}{Q_{A}+Q_{B}}}}$$
</center>

where $${\displaystyle Q_{A}=10^{R_{A}/400}}$$ and  $${\displaystyle Q_{B}=10^{R_{B}/400}}$$.

<br>
<br>
A player whose rating is 100 points greater than their opponents is expected to win 64% of the time; if the difference is 200 points, then the expected probability of winning for the stronger player is 76%.  This property of the Elo system allows us to compare fighters of different generations in fantasy match-ups.


## Data

Boxrec is one source of historical boxing records, though incomplete, it is a good source for historical rankings, fighter biographical data, and match outcomes.  Boxrec even has their own ranking system, which is described here.  For this analysis, I used data from Boxrec to calculate the various rankings.

| Rank  |      Fighter     |  Elo Max |
|:----------:|:-------------:|:------:|
|1. |Sugar Ray Robinson|2069.52|
|2.|Harry Greb|2052.88|
|3. |Benny Leonard|2019.76| 
|4. |Henry Armstrong|2010.28| 
|5. |Willie Pep|1978.91|
|6. |Joe Louis|1975.84|
|7. |Archie Moore|1975.61| 
|8. |Freddie Steele| 1963.16| 
|9. |Ezzard Charles| 1962.89| 
|10.|Barney Ross| 1947.24| 
|11.|Ike Williams| 1946.65|
|12.|Kid Gavilan| 1933.69| 
|13. |Freddie Miller| 1933.29|
|14. |Tony Canzoneri| 1932.09| 
|15. |Mickey Walker| 1924.69|
|16. |Lou Ambers| 1923.47|
|17. |Carlos Monzon| 1920.35|
|18. |Tommy Loughran| 1918.15|
|19. |Young Corbett III|1916.57|
|20. |Muhammad Ali|1909.32|
|21. |Ceferino Garcia|1907.26|
|22. |Jose Napoles|1900.59|
|23. |Bobo Olson|1899.97|
|24. |Duilio Loi|1898.70|
|25. |Maxie Rosenbloom|1897.43|




| Rank  |      Fighter     |  Elo Average |
|:----------:|:-------------:|:------:|
|1. |Sugar Ray Robinson| 1855.00|
|2. |Harry Greb| 1817.61|
|3. |Willie Pep| 1806.41|
|4. |Henry Armstrong| 1759.86|
|5. |Archie Moore| 1756.91|
|6. |Joe Louis| 1739.19| 
|7. |Holman Williams| 1732.01|
|8. |Maxie Rosenbloom| 1723.70|
|9. |Luis Manuel Rodriguez| 1721.78| 
|10. |Tony Canzoneri| 1719.02|
|11. |Freddie Miller| 1716.27|
|12. |Ezzard Charles| 1715.72|
|13.|Lew Tendler| 1712.23|
|14. |Kid Gavilan| 1710.57|
|15. |Kid Chocolate| 1703.23|
|16. |Young Stribling| 1702.78|
|17. |Ike Williams| 1701.01|
|18. |Benny Leonard| 1699.89|
|19.  |Emile Griffith| 1699.42|
|20. |Lou Ambers| 1689.37|
|21. |Jackie Fields| 1687.93|
|22. |Tommy Gibbons| 1687.08|
|23. |Bobo Olson| 1685.10|
|24. |Wesley Ramey| 1684.10| 
|25.  |Muhammad Ali| 1682.48|




Highest Rated Bouts:

|Fighter| Elo | Record | Opponent | Elo | Record | Date|
|:----------:|:-------------:|:------:|:----------:|:------:|:------:|:---------:|
|Jake LaMotta | 1845.24 |78-14-3| Sugar Ray Robinson (W,TKO) | 2058.42|120-1-2 | 1951-02-14|  
|Bobo Olson | 1885.60  |71-8-0|Sugar Ray Robinson (W,KO) | 2014.54|137-4-2|1956-05-18  |
|Sugar Ray Robinson (W,KO) | 2003.29    |136-4-2|      Bobo Olson | 1896.84 |71-7-0| 1955-12-09  |
|Joe Louis | 1966.90   |58-1-0|   Ezzard Charles (W,UD) | 1918.10|66-5-1|1950-09-27|
|Henry Armstrong (D,PTS) | 1987.04|106-12-6|    Ceferino Garcia | 1896.12|112-25-12|1940-03-01 |
  

Worst Mismatches:

|Fighter| Elo | Record | Opponent | Elo | Record |
|:----------:|:-------------:|:------:|:----------:|:------:|:------:|
|Al Benedict|  985.94 |10-26-0|Harry Greb (W,TKO) | 1836.32  |61-2-2|
|Willie Pep (W,PTS)| 1898.36|161-4-1|Kenny Leach | 1040.90 |3-14-0|  
|Willie Pep (W,PTS)| 1872.37|185-6-1| Mario Eladio Colon | 1043.31  |15-47-8|
|Johnny Cockfield |10-84-9|  867.80  | Chalky Wright (W,TKO) | 1774.65 |  153-34-18|
|Archie Moore (W,RTD) | 1967.89    |178-21-9|     George Abinet  |1073.68|6-14-1|  



