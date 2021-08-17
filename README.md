# NYC Citibike Bicyle Longevity Analysis (2013-2021)

The NYC Citibike program releases a large amount of data on how riders use the service. Since 2013, more than 112 million trips have been taken. This analysis takes a deeper look at the bikes in the Citibike fleet, and how well they have withstood the test of time.

## Tools Used
Python, pandas, and SQLalchemy were used to Perform ETL operations and query the resultant SQLite database. 

Tableau provided the framework for making an interactive storyboard.


## Analysis

Since Citibike began in 2013, more than 32,000 unique bike IDs have been observed in the publc data. As of January 2021, when Citibike stopped sharing information on bike ID, there were only 14,000 of those bikes currently in use.

[<img src="https://github.com/bakerv/citibike-data-extraction_2013-2021/blob/main/images/overall_fleet_status.PNG">](href="href="https://public.tableau.com/app/profile/victor.baker/viz/NYCCitibikeBicycleLongevityAnalysis2013-2021/NYCCitibikeBicycleLongevityAnalysis")


Bike longevity varies between cohorts. The initial cohort, released in 2013, had an observed lifespan of 5.8 years and nearly 1800 riding hours. The majority of the bikes were still in service in 2020, when they were replaced by 8000 brand new bikes. In contrast to this sucess story is 2015, the worst performing cohort of bikes to date. These 2500 bikes barely lasted a year, and only saw an average of 600 riding hours. In fact, no cohort since 2016 has a lifespan of more than 2 years. [Check out the dashboard](href="https://public.tableau.com/app/profile/victor.baker/viz/NYCCitibikeBicycleLongevityAnalysis2013-2021/NYCCitibikeBicycleLongevityAnalysis") to dive further into the cohort statistics. 

 What does this mean for the current phase 3 expansion?
- With Citibike and Lyft increasing the size of the bike fleet to 40,000 bikes, expect a signifigant amount of attrition to come with that process.

- At the end of the 4 year 28,000 bike expansion, another 28,000 (or more) bikes will be needed to maintain the fleet at that size.

## Limitations

There is no way for us to know why bikes have been removed from service. Citibike does not release information on bike maintenance activities, and removed tracking data for individual bikes in February 2021.

We do not know if bike ID changed at any point for a given bike. Were those 2500 short-lived bikes from 2015 refurbished an released with new IDs? Did they get scrapped for parts for that long lived 2013 cohort? Have lessons learned been implented to enusre a longer lasting future fleet?

 Further insight from Citibike is needed to answer these questions.




