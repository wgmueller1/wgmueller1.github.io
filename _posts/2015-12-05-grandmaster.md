---
layout: post
title: Boxings Grandmasters
description: "Analysis of Historical boxing data"
modified: 2015-12-05
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


As a lifelong boxing fan and someone whose everyday job involves analyzing massive amounts of data, I thought it would be interesting to apply some quantitative analysis to a sport which has many subjective aspects to it.

##ELO rating system

The Elo rating system is a method for calculating the relative skill levels of players in competitor-versus-competitor games.   The difference in the ratings between two players serves as a predictor of the outcome of a match. Two players with equal ratings who play against each other are expected to score an equal number of wins. A player whose rating is 100 points greater than their opponents is expected to score 64%; if the difference is 200 points, then the expected score for the stronger player is 76%.



##Data

Boxrec is one source of historical boxing records, though incomplete, it is a good source for historical rankings, fighter biographical data, and match outcomes.  Boxrec even has their own ranking system, which is described here.  For this analysis, I used data from Boxrec to calculate the various rankings.

| Rank  |      Fighter     |  Elo Max |
|:----------:|:-------------:|:------:|
|1. |Sugar Ray Robinson|2069.520867275613|
|2.|Harry Greb|2052.8763634156262|
|3. |Benny Leonard|2019.7584089617503| 
|4. |Henry Armstrong|2010.2763179787501| 
|5. |Willie Pep|1978.9129880799419|
|6. |Joe Louis|1975.8391908889255|
|7. |Archie Moore|1975.610967270421| 
|8. |Freddie Steele| 1963.1645149271533| 
|9. |Ezzard Charles| 1962.8868382295352| 
|10.|Barney Ross| 1947.2447907846429| 
|11.|Ike Williams| 1946.6548133928763|
|12.|Kid Gavilan| 1933.6989348611974| 
|13. |Freddie Miller| 1933.2904640109389|
|14. |Tony Canzoneri| 1932.0925443629635| 
|15. |Mickey Walker| 1924.6891290386548|
|16. |Lou Ambers| 1923.4732567233016|
|17. |Carlos Monzon| 1920.3524175170619|
|18. |Tommy Loughran| 1918.1457652903396|
|19. |Young Corbett III|1916.5742488712126|
|20. |Muhammad Ali|1909.3184663333086|
|21. |Ceferino Garcia|1907.2591348209057|
|22. |Jose Napoles|1900.5897523420126|
|23. |Bobo Olson|1899.9746585711175|
|24. |Duilio Loi|1898.7037664713662|
|25. |Maxie Rosenbloom|1897.4312977331319|




| Rank  |      Fighter     |  Elo Average |
|:----------:|:-------------:|:------:|
|1. |Sugar Ray Robinson| 1855.0036761450069|
|2. |Harry Greb| 1817.6155130555853|
|3. |Willie Pep| 1806.4191626922304|
|4. |Henry Armstrong| 1759.8612501580649|
|5. |Archie Moore| 1756.9155873311097|
|6. |Joe Louis| 1739.1950176641708| 
|7. |Holman Williams| 1732.0182657937196|
|8. |Maxie Rosenbloom| 1723.7008910856403|
|9. |Luis Manuel Rodriguez| 1721.7803226184903| 
|10. |Tony Canzoneri| 1719.0282741224114|
|11. |Freddie Miller| 1716.2742626938314|
|12. |Ezzard Charles| 1715.7150720314391|
|13.|Lew Tendler| 1712.2352900318194|
|14. |Kid Gavilan| 1710.5764454380462|
|15. |Kid Chocolate| 1703.2289637348927|
|16. |Young Stribling| 1702.7833099927777|
|17. |Ike Williams| 1701.0128321834193|
|18. |Benny Leonard| 1699.8913675280237|
|19.  |Emile Griffith| 1699.4161846718416|
|20. |Lou Ambers| 1689.3753983513191|
|21. |Jackie Fields| 1687.9290334844452|
|22. |Tommy Gibbons| 1687.0751035847591|
|23. |Bobo Olson| 1685.1010072493598|
|24. |Wesley Ramey| 1684.1023652026458| 
|25.  |Muhammad Ali| 1682.4794435437068|




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



