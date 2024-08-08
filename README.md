# Dataset Title: Formula 1 World Championship (1950 - 2024)

This dataset is sourced from Kaggle and was compiled by Rohan Rao. It contains comprehensive data on Formula 1 races, teams, drivers, and circuits spanning from the inception of the World Championship in 1950 up until 2020. The dataset is available at

[ Formula 1 World Championship (1950 - 2024).]([https://8weeksqlchallenge.com](https://www.kaggle.com/datasets/rohanrao/formula-1-world-championship-1950-2020/data)). 


Dataset Contents:

**The dataset is structured across multiple CSV files, each representing different aspects of Formula 1 data:** 

 - Drivers: Information on F1 drivers, including their names, nationalities, and birthdates.
 - Constructors: Data on F1 teams (constructors), including their names and nationalities.
 - Circuits: Details of race circuits, including location, country, and other geographical data.
 - Races: A comprehensive record of every race, including the year, round, and circuit where it was held.
 - Results: Detailed results of each race, covering the position of drivers, the constructor, and points awarded.
 - Driver Standings: Annual standings for each driver in the championship, detailing their rank and points.
 - Constructor Standings: Annual standings for each constructor, showing their rank and points.
 - Lap Times: Lap-by-lap times for each driver, detailing their performance during races.
 - Pit Stops: Information on pit stops made during races, including timings and duration.
 - Qualifying: Data on the qualifying sessions, including positions and times.
 - Seasons: A summary of each F1 season from 1950 to 2020, including the number of races.


## Description and Motivation: ##


As a passionate Formula 1 fan who has followed every race since 2020, even attending live events, you aim to combine your enthusiasm for the sport with your desire to learn and practice SQL skills. This dataset provides a rich and diverse foundation for exploring various aspects of Formula 1, allowing you to analyze historical trends, driver and constructor performance, race outcomes, and much more.


## Entity Relationship Diagram

![Untitled](https://github.com/user-attachments/assets/0c43d2d9-e779-4f21-a6c5-4ee3c67186f1)


## Questions and answers

#### 1. Driver Positions View (drivers_positions):

 Creates a view to centralize information about races, drivers, and their performance, combining race results with driver and constructor details.

 ```sql

DROP VIEW drivers_positions;

CREATE VIEW drivers_positions
AS
  SELECT r.raceid         AS race_id,
         r.circuitid      AS circuit_id,
         r.year           AS year,
         r.NAME           AS NAME,
         re.driverid      AS driver_id,
         re.constructorid AS constructor_id,
         re.positionorder AS finish_position,
         re.grid          AS start_position,
         re.statusid      AS status_id,
         re.rank          AS rank,
         d.forename
         || ' '
         || d.surname     AS driver,
         d.nationality    AS nationality
  FROM   races AS r
         LEFT JOIN results AS re
                ON re.raceid = r.raceid
         LEFT JOIN drivers AS d
                ON d.driverid = re.driverid;

```


-------------

#### 2. Average Finish Position::

 Calculates the average finishing and starting positions of drivers, highlighting those who consistently finish far from the front (offset by 200 and fetch last 20).

 ```sql

SELECT   driver,
         Round(Avg(finish_position),2)                                AS avg_finish,
         Round(Avg(start_position),2)                                 AS position_on_start,
         Round(Avg(start_position),2) - Round(Avg(finish_position),2) AS avg_position_gained_from_start,
         Count(*)                                                     AS races_finished,
         Rank() over (ORDER BY round(avg(finish_position),2) ASC)     AS ranking
FROM     drivers_positions
WHERE    status_id IN (1,11,12,13)
GROUP BY driver
HAVING   count(*) > 10
ORDER BY avg_finish ASC
OFFSET   200
FETCH next 20 rows only;
```



-------------

#### 3. Percentage of Wins (Top 20):

Calculates the percentage of wins out of total races finished for drivers with at least 4 races completed, ranking the top 20 drivers by win percentage.

 ```sql

SELECT driver,
       Sum (CASE
              WHEN finish_position = 1 THEN 1
              ELSE 0
            END)                                                      AS
       number_of_wins,
       COUNT(*)                                                       AS
       total_finished,
       Round(( Sum (CASE
                      WHEN finish_position = 1 THEN 1
                      ELSE 0
                    END) :: numeric / COUNT(*) :: numeric ) * 100, 2) AS
       percentage_wins
FROM   drivers_positions
WHERE  status_id IN ( 1, 11, 12, 13 )
GROUP  BY driver
HAVING COUNT(*) >= 4
ORDER  BY percentage_wins desc
FETCH FIRST 20 ROWS ONLY; 
```


-------------

#### 4. Top Drivers by Wins:

Lists drivers with the most race wins, applying a dense rank to handle ties and fetching the top 25 drivers.

 ```sql

SELECT driver                      AS driver,
       COUNT(*)                    AS wins_total,
       DENSE_RANK()
         OVER (
           ORDER BY COUNT(*) desc) AS ranking
FROM   drivers_positions
WHERE  finish_position = 1
GROUP  BY driver
ORDER  BY COUNT(*) desc
FETCH FIRST 25 ROWS only; 
```

-------------

#### 5. Most Podiums:

Identifies drivers with the most podium finishes (1st, 2nd, or 3rd place) and ranks them, showing the top 25.

 ```sql
SELECT driver                      AS driver,
       COUNT(*)                    AS podiums,
       DENSE_RANK()
         OVER (
           ORDER BY COUNT(*) desc) AS rankingD
FROM   drivers_positions
WHERE  finish_position IN ( 1, 2, 3 )
GROUP  BY driver
ORDER  BY COUNT(*) desc
FETCH first 25 ROWS only; 
```
-------------


-------------


#### 6. 2007 Constructor Championship Adjustment::
Adjusts the constructor championship totals for McLaren and Ferrari based on the 2007 season because McLaren was disqualified from constructor championship that year 

 ```sql
SELECT team_name,
       CASE
         WHEN team_name = 'McLaren' THEN total_championship - 1
         WHEN team_name = 'Ferrari' THEN total_championship + 1
         ELSE total_championship
       END AS constructor_championship
FROM   (SELECT team_name,
               Sum(CASE
                     WHEN ranking = 1 THEN 1
                     ELSE 0
                   END) AS total_championship
        FROM   (SELECT r.year                            AS year,
                       c.NAME                            AS team_name,
                       Max(cs.points)                    AS points,
                       Rank()
                         OVER (
                           partition BY r.year
                           ORDER BY Max(cs.points) DESC) AS ranking
                FROM   constructor_standings AS cs
                       LEFT JOIN constructors AS c
                              ON c.constructorid = cs.constructorid
                       LEFT JOIN races AS r
                              ON r.raceid = cs.raceid
                GROUP  BY c.NAME,
                          r.year
                ORDER  BY r.year) AS sub
        GROUP  BY team_name
        ORDER  BY total_championship DESC) AS sub2
ORDER  BY constructor_championship DESC; 
```
-------------


#### 7. Race Wins by Driver and Win Percentage (year):
For each driver, it calculates the total wins and win percentage for each season, along with pole positions, considering drivers who participated in at least 60% of the season's races.

 ```sql
SELECT driver,
       year,
       SUM(CASE
             WHEN finish_position = 1 THEN 1
             ELSE 0
           END)                                            AS wins_by_season,
       Round(SUM(CASE
                   WHEN finish_position = 1 THEN 1
                   ELSE 0
                 END) :: NUMERIC / Count(*) :: NUMERIC, 2) AS
       wins_percentage_by_season,
       SUM(CASE
             WHEN start_position = 1 THEN 1
             ELSE 0
           END)
       pole_positions_by_season,
       Round(SUM(CASE
                   WHEN start_position = 1 THEN 1
                   ELSE 0
                 END) :: NUMERIC / Count(*) :: NUMERIC, 2) AS
       pole_positions_by_season,
       Count(*)                                            AS races
FROM   drivers_positions AS dp
GROUP  BY year,
          driver
HAVING Count(*) >= 0.5 * (SELECT Count(DISTINCT race_id)
                          FROM   drivers_positions
                          WHERE  year = dp.year) :: NUMERIC
ORDER  BY wins_percentage_by_season DESC; 

```
-------------

#### 8. Most Successful Driver by Country:
For each driver, it calculates the total wins and win percentage for each season, along with pole positions, considering drivers who participated in at least 50% of the season's races.

 ```sql
SELECT *
FROM   (SELECT driver,
               nationality,
               Dense_rank()
                 OVER (
                   partition BY nationality
                   ORDER BY Count(*) DESC) AS ranking,
               Count(*)                    AS wins
        FROM   drivers_positions
        WHERE  finish_position = 1
        GROUP  BY nationality,
                  driver) AS sub
WHERE  ranking = 1
ORDER  BY wins DESC; 

```
-------------

#### 9. Most Successful Nations:
Aggregates race results to determine which countries have the most wins, podiums, and overall successful races.

 ```sql
SELECT   nationality,
         Sum(
         CASE
                  WHEN finish_position = 1 THEN 1
                  ELSE 0
         END) AS first_place,
         Sum(
         CASE
                  WHEN finish_position = 2 THEN 1
                  ELSE 0
         END) AS second_place,
         Sum(
         CASE
                  WHEN finish_position = 3 THEN 1
                  ELSE 0
         END) AS third_place,
         Count(*) filter(WHERE finish_position IN(1,2,3)
AND      status_id = 1) AS total
FROM     drivers_positions
WHERE    status_id = 1
GROUP BY nationality
ORDER BY first_place DESC,
         second_place DESC,
         total DESC;



```
-------------

#### 10. Drivers Who Won a Grand Prix (Top 5 Countries):
Lists drivers from the top 5 countries with the most grand prix wins, summarizing the total wins and listing drivers from each country.

 ```sql
SELECT   sub.nationality,
         Sum(sub.wins) AS wins,
         string_agg(sub.driver,', ' order BY sub.wins)
FROM     (
                  SELECT   nationality,
                           driver,
                           sum(
                           CASE
                                    WHEN finish_position = 1 THEN 1
                                    ELSE 0
                           END) AS wins
                  FROM     drivers_positions
                  WHERE    status_id = 1
                  GROUP BY nationality,
                           driver) AS sub
WHERE    wins > 0
GROUP BY sub.nationality
ORDER BY wins DESC offset 0FETCH first 5 rows only;

```
-------------


#### 11. Drivers Who Won a Grand Prix grouped by country with total wins by country:
Lists drivers who have won grand prix and their's nationality with total win by country

 ```sql
SELECT COALESCE(nationality, 'PODIUMS TOTAL'),
       COALESCE(driver, 'COUNTRY TOTAL'),
       Sum(CASE
             WHEN finish_position = 1 THEN 1
             ELSE 0
           END) AS WINS
FROM   drivers_positions
WHERE  status_id = 1
GROUP  BY rollup( nationality, driver )
HAVING Sum(CASE
             WHEN finish_position = 1 THEN 1
             ELSE 0
           END) > 0
ORDER  BY nationality,
          wins DESC; 

```
-------------


#### 12. Drivers with Most Races without a Win:
Identifies drivers who have participated in the most races without securing a win, excluding disqualified races.

 ```sql
SELECT driver,
       Count(*) AS races_without_win
FROM   drivers_positions
WHERE  driver_id NOT IN (SELECT DISTINCT driver_id
                         FROM   drivers_positions
                         WHERE  status_id IN ( 1, 11, 12, 13 )
                                AND finish_position = 1)
       AND status_id != 2
GROUP  BY driver
ORDER  BY Count(*) DESC; 

```
-------------

#### 13. Drivers with Most Races without a Podium:
Similar to the previous query but focuses on drivers who have never reached the podium, dsq excluded

 ```sql
SELECT driver,
       Count(*) AS races_without_win
FROM   drivers_positions
WHERE  driver_id NOT IN (SELECT DISTINCT driver_id
                         FROM   drivers_positions
                         WHERE  status_id IN ( 1, 11, 12, 13 )
                                AND finish_position = 1)
       AND status_id != 2
GROUP  BY driver
ORDER  BY Count(*) DESC; 

```
-------------


#### 14. Drivers with Most DNFs:
Analyzes which drivers have failed to finish (DNF) the most races, showing drivers with more than 7 race starts.


 ```sql
-- approx data is not accurate due to status_id
WITH races_unfinished
     AS (SELECT driver_id,
                driver,
                Count(DISTINCT race_id) AS races_did_not_finish
         FROM   drivers_positions AS dp
                left join results AS r
                       ON dp.driver_id = r.driverid
                          AND dp.race_id = r.raceid
         WHERE  status_id NOT IN ( 1, 11, 12, 13 )
         GROUP  BY driver_id,
                   driver),
     total_races
     AS (SELECT driver_id,
                driver,
                Count(*) AS races_total
         FROM   drivers_positions
         WHERE  driver_id IS NOT NULL
         GROUP  BY driver_id,
                   driver)
SELECT total_races.driver_id,
       total_races.driver,
       races_did_not_finish,
       races_total,
       Round(( races_did_not_finish :: NUMERIC / races_total :: NUMERIC ) * 100,
       2) AS
       percentage_unfinished
FROM   total_races
       left join races_unfinished
              ON total_races.driver_id = races_unfinished.driver_id
WHERE  races_total > 7
ORDER  BY races_did_not_finish DESC; 

```
-------------


#### 15. Pole to Win Ratio::
Calculates the success rate of converting pole positions into race wins, for drivers with at least 2 wins from pole position.


 ```sql
SELECT driver_id,
       driver,
       Round(SUM(CASE
                   WHEN start_position = 1
                        AND finish_position = 1 THEN 1
                   ELSE 0
                 END) :: NUMERIC / SUM(CASE
                                         WHEN start_position = 1 THEN 1
                                         ELSE 0
                                       END) :: NUMERIC * 100, 2) AS pole_to_win
FROM   drivers_positions
WHERE  driver_id IN (SELECT driver_id
                     FROM   drivers_positions
                     WHERE  start_position = 1
                     GROUP  BY driver_id)
GROUP  BY driver_id,
          driver
HAVING SUM(CASE
             WHEN start_position = 1
                  AND finish_position = 1 THEN 1
             ELSE 0
           END) >= 2
ORDER  BY pole_to_win DESC; 


```
-------------


#### 16. Most Visited Tracks:
Lists the top 25 circuits by the number of races held.


 ```sql
SELECT r.name,
       COUNT(*) AS total_races
FROM   races AS r
       LEFT JOIN circuits AS ci
              ON r.circuitid = ci.circuitid
GROUP  BY r.name
ORDER  BY total_races desc
FETCH first 25 ROWS only; 


```
-------------


#### 17. Top 10 Teams by Wins:
Identifies the top 10 constructors by the total number of race wins.

 ```sql
SELECT CON.constructorid AS CONSTRUCTOR_ID,
       CON.NAME          AS TEAM,
       Count(*)          AS WINS
FROM   drivers_positions AS DP
       LEFT JOIN constructors AS CON
              ON DP.constructor_id = CON.constructorid
WHERE  finish_position = 1
GROUP  BY CON.constructorid,
          CON.NAME
ORDER  BY wins DESC; 


```
-------------


#### 18. Constructor with Most 1-2 Finishes:
Determines which constructor has the most 1-2 finishes in races (both drivers finishing in the top 2), and calculates the percentage of these finishes relative to total wins.

 ```sql
WITH p1_p2_finishes AS
(
           SELECT     dp1.race_id AS race_id,
                      dp1.year    AS year,
                      dp1.driver,
                      dp1.finish_position AS driver1_position,
                      dp2.driver,
                      dp2.finish_position AS driver2_position,
                      dp1.constructor_id  AS constructor_id
           FROM       drivers_positions   AS dp1
           INNER JOIN drivers_positions   AS dp2
           ON         dp1.race_id = dp2.race_id
           AND        dp1.constructor_id = dp2.constructor_id
           AND        dp1.driver > dp2.driver
           WHERE      (
                                 dp1.finish_position <=2
                      AND        dp1.status_id =1)
           AND        (
                                 dp2.finish_position <=2
                      AND        dp2.status_id = 1)), wins AS
(
          SELECT    con.constructorid AS constructor_id,
                    con.NAME          AS team,
                    Count(*)          AS wins
          FROM      drivers_positions AS dp
          LEFT JOIN constructors      AS con
          ON        dp.constructor_id = con.constructorid
          WHERE     finish_position = 1
          GROUP BY  con.constructorid,
                    con.NAME
          ORDER BY  wins DESC)
SELECT     w.team AS team_name,
           w.wins,
           Count(DISTINCT f.race_id)                                         AS p1_p2_nbr,
           Round((Count(DISTINCT f.race_id)::numeric/w.wins::numeric)*100,1) AS per_of_p1_p2
FROM       p1_p2_finishes                                                    AS f
RIGHT JOIN wins                                                              AS w
ON         f.constructor_id = w.constructor_id
GROUP BY   w.team,
           w.wins
ORDER BY   wins DESC,
           per_of_p1_p2 offset 0FETCH first 15 rows only;



```
-------------


#### 19. Teammate Slayer:
Determines the percentage of times a driver finished ahead of their teammate, only considering races with exactly two drivers per team and excluding cases where both drivers had the same finish position.

 ```sql
WITH p1_p2_finishes AS
(
           SELECT     dp1.race_id AS race_id,
                      dp1.year    AS year,
                      dp1.driver,
                      dp1.finish_position AS driver1_position,
                      dp2.driver,
                      dp2.finish_position AS driver2_position,
                      dp1.constructor_id  AS constructor_id
           FROM       drivers_positions   AS dp1
           INNER JOIN drivers_positions   AS dp2
           ON         dp1.race_id = dp2.race_id
           AND        dp1.constructor_id = dp2.constructor_id
           AND        dp1.driver > dp2.driver
           WHERE      (
                                 dp1.finish_position <=2
                      AND        dp1.status_id =1)
           AND        (
                                 dp2.finish_position <=2
                      AND        dp2.status_id = 1)), wins AS
(
          SELECT    con.constructorid AS constructor_id,
                    con.NAME          AS team,
                    Count(*)          AS wins
          FROM      drivers_positions AS dp
          LEFT JOIN constructors      AS con
          ON        dp.constructor_id = con.constructorid
          WHERE     finish_position = 1
          GROUP BY  con.constructorid,
                    con.NAME
          ORDER BY  wins DESC)
SELECT     w.team AS team_name,
           w.wins,
           Count(DISTINCT f.race_id)                                         AS p1_p2_nbr,
           Round((Count(DISTINCT f.race_id)::numeric/w.wins::numeric)*100,1) AS per_of_p1_p2
FROM       p1_p2_finishes                                                    AS f
RIGHT JOIN wins                                                              AS w
ON         f.constructor_id = w.constructor_id
GROUP BY   w.team,
           w.wins
ORDER BY   wins DESC,
           per_of_p1_p2 offset 0FETCH first 15 rows only;


```
-------------


#### 20. First and Last Race Winner for Each Circuit:
Finds the first and last driver to win at each circuit, providing a historical overview of race winners.

 ```sql
WITH first_winner AS (
  SELECT 
    DISTINCT ON(c.NAME) C.NAME, 
    driver, 
    year 
  FROM 
    drivers_positions AS DP 
    LEFT JOIN circuits AS C ON DP.circuit_id = C.circuitid 
  WHERE 
    finish_position = 1 
  ORDER BY 
    C.NAME, 
    year ASC
), 
race_count AS (
  SELECT 
    C.NAME, 
    Count(DISTINCT circuit_id) AS RACE_OCCURRENCES 
  FROM 
    drivers_positions AS DP 
    LEFT JOIN circuits AS C ON DP.circuit_id = C.circuitid 
  GROUP BY 
    C.NAME 
  HAVING 
    Count(DISTINCT race_id) > 1
), 
last_winner AS (
  SELECT 
    DISTINCT ON(c.NAME) C.NAME, 
    driver, 
    year 
  FROM 
    drivers_positions AS DP 
    LEFT JOIN circuits AS C ON DP.circuit_id = C.circuitid 
    INNER JOIN race_count AS RC ON RC.NAME = C.NAME 
  WHERE 
    finish_position = 1 
  ORDER BY 
    C.NAME, 
    year DESC
) 
SELECT 
  FW.NAME, 
  FW.driver AS FIRST_WINNER, 
  FW.year AS YEAR_WON, 
  LW.driver AS LAST_WINNER, 
  LW.year AS YEAR_WON 
FROM 
  first_winner AS FW 
  LEFT JOIN last_winner AS LW ON FW.NAME = LW.NAME;


```
-------------


#### 21. Constructor Standings in 2022 (Including Crosstab):
Tracks constructor standings round by round for the 2022 season, with a crosstab format to compare points across all rounds.

 ```sql
SELECT 
  C.name AS team, 
  Any_value(CS.points) AS POINTS_SCORED, 
  Rank() over (
    PARTITION BY R.DATE 
    ORDER BY 
      Any_value(CS.points) DESC
  ) :: INTEGER AS RANKING, 
  round 
FROM 
  constructor_standings AS CS 
  left join races AS R ON CS.raceid = R.raceid 
  left join constructors AS C ON CS.constructorid = C.constructorid 
WHERE 
  year = 2022 
GROUP BY 
  C.name, 
  R.DATE, 
  round 
ORDER BY 
  R.DATE, 
  Any_value(CS.points) DESC;


```

With crosstab


 ```sql
CREATE EXTENSION IF NOT EXISTS tablefunc;
-- Create the crosstab query
SELECT 
  * 
FROM 
  crosstab(
    $$SELECT C.NAME AS team, 
    R.ROUND, 
    ANY_VALUE(CS.POINTS) AS POINTS_SCORED 
    FROM 
      CONSTRUCTOR_STANDINGS AS CS 
      LEFT JOIN RACES AS R ON CS.RACEID = R.RACEID 
      LEFT JOIN CONSTRUCTORS AS C ON CS.CONSTRUCTORID = C.CONSTRUCTORID 
    WHERE 
      R.YEAR = 2022 
    GROUP BY 
      C.NAME, 
      R.ROUND 
    ORDER BY 
      C.NAME, 
      R.ROUND$$, 
      $$SELECT DISTINCT ROUND 
    FROM 
      RACES 
    WHERE 
      YEAR = 2022 
    ORDER BY 
      ROUND$$
  ) AS ct (
    team TEXT, "Round 1" INT, "Round 2" INT, 
    "Round 3" INT, "Round 4" INT, "Round 5" INT, 
    "Round 6" INT, "Round 7" INT, "Round 8" INT, 
    "Round 9" INT, "Round 10" INT, "Round 11" INT, 
    "Round 12" INT, "Round 13" INT, "Round 14" INT, 
    "Round 15" INT, "Round 16" INT, "Round 17" INT, 
    "Round 18" INT, "Round 19" INT, "Round 20" INT, 
    "Round 21" INT, "Round 22" INT
  ) 
ORDER BY 
  "Round 22" DESC;



```
-------------


#### 23. Driver Standings in 2021 Season (Including Crosstab):
Analyzes driver standings across the 2021 season, providing a crosstab to show cumulative points and rankings round by round.
 ```sql
SELECT 
  CODE, 
  SUM(POINTS_PER_ROUND) OVER(
    PARTITION BY CODE 
    ORDER BY 
      ROUND_NBR
  ), 
  POINTS_PER_ROUND, 
  ROUND_NBR, 
  RANK() OVER (
    PARTITION BY ROUND_NBR 
    ORDER BY 
      CUMULATIVE_SUM DESC
  ) 
FROM 
  (
    SELECT 
      CODE, 
      SUM(R.POINTS) OVER(
        PARTITION BY CODE, 
        ROUND 
        ORDER BY 
          RA.ROUND
      ) as POINTS_PER_ROUND, 
      SUM(R.POINTS) OVER(
        PARTITION BY CODE 
        ORDER BY 
          RA.ROUND
      ) AS CUMULATIVE_SUM, 
      RA.ROUND AS ROUND_NBR 
    FROM 
      RESULTS AS R 
      LEFT JOIN DRIVERS AS D ON D.DRIVERID = R.DRIVERID 
      LEFT JOIN RACES AS RA ON R.RACEID = RA.RACEID 
    WHERE 
      RA.YEAR = 2021
  ) AS SUB

```


With crosstab


 ```sql
SELECT 
  * 
FROM 
  crosstab(
    $$SELECT CODE, 
    ROUND_NBR, 
    CUMULATIVE_SUM || ' (Rank: ' || RANKING || ')' AS VALUE 
    FROM 
      (
        SELECT 
          CODE, 
          ROUND_NBR, 
          CUMULATIVE_SUM, 
          RANK() OVER (
            PARTITION BY ROUND_NBR 
            ORDER BY 
              CUMULATIVE_SUM DESC
          ) AS RANKING 
        FROM 
          (
            SELECT 
              CODE, 
              SUM(R.POINTS) OVER (
                PARTITION BY CODE, 
                RA.ROUND 
                ORDER BY 
                  RA.ROUND
              ) AS POINTS_PER_ROUND, 
              SUM(R.POINTS) OVER (
                PARTITION BY CODE 
                ORDER BY 
                  RA.ROUND
              ) AS CUMULATIVE_SUM, 
              RA.ROUND AS ROUND_NBR 
            FROM 
              RESULTS AS R 
              LEFT JOIN DRIVERS AS D ON D.DRIVERID = R.DRIVERID 
              LEFT JOIN RACES AS RA ON R.RACEID = RA.RACEID 
            WHERE 
              RA.YEAR = 2021
          ) AS SUB
      ) AS FINAL 
    ORDER BY 
      CODE, 
      ROUND_NBR, 
      CUMULATIVE_SUM DESC $$, 
      $$SELECT DISTINCT ROUND AS ROUND_NBR 
    FROM 
      RACES 
    WHERE 
      YEAR = 2021 
    ORDER BY 
      ROUND$$
  ) AS ct (
    CODE TEXT, "Round 1" TEXT, "Round 2" TEXT, 
    "Round 3" TEXT, "Round 4" TEXT, "Round 5" TEXT, 
    "Round 6" TEXT, "Round 7" TEXT, "Round 8" TEXT, 
    "Round 9" TEXT, "Round 10" TEXT, "Round 11" TEXT, 
    "Round 12" TEXT, "Round 13" TEXT, 
    "Round 14" TEXT, "Round 15" TEXT, 
    "Round 16" TEXT, "Round 17" TEXT, 
    "Round 18" TEXT, "Round 19" TEXT, 
    "Round 20" TEXT, "Round 21" TEXT, 
    "Round 22" TEXT
  ) 
ORDER BY 
  CAST(
    SPLIT_PART("Round 22", ' ', 1) AS NUMERIC
  ) DESC NULLS LAST;



```
-------------


























