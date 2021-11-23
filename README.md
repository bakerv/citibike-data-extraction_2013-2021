# NYC Citibike Bicyle Longevity Analysis (2013-2021)

The NYC Citibike program is the largest bike share program in the Nation. After seeing a spike in demand for the services in 2019, it was announced that the program aims to have 40,000 bikes in service by 2023. This analysis exmaines the lifepsan of bikes in the Citibike fleet, and draws conclusions about many new bikes will it will take to reach that goal.

### Tools Used
Python, Pandas, and SQLAlchemy  - Perform ETL operations and query the resultant SQLite database.

ZipFile, io - Extract the file contents into memory and minimize disk usage.

Tableau - Data Visualiztion and dashboard creation.


## SQL Database Creation
First thing, we need to import our dependencies. This script in written in Python version 3.8.11.

```Python
import io
import pandas as pd
import requests
import sqlalchemy
import zipfile
```
The next step in the workflow is to access the data, and create a database we can query from. 
The data from the Citibike API comes on the form of zipped .csv files stored in an AWS S3 Bucket. Glancing through the index, we see that the file extension change from .zip to .csv.zip in January of 2017.



To handle this, we use list comprehension to generate a list containing the file names.

```python
years = ['2013','2014','2015','2016','2017', '2018','2019', '2020']
months = ['01','02','03','04','05','06','07','08','09','10','11','12']
file_names = [f'{year}{month}-citibike-tripdata' for year in years for month in months]
file_names.append('202101-citibike-tripdata')
```

Note: Citibike stoped sharing tracking information on bike ID in February of 2021. Because of this, we will not be able to examine lifespans beyond this point.

At this point we will also use SQLAlchemy to create a connection to a location SQLite database. This will be used later on in the workflow to store clean data from each file.

```Python
save_directory = '../../../Data_Sets/citibike/'
db_name = '201306-202101-citibike-tripdata.sqlite'
engine = sqlalchemy.create_engine(f'sqlite:///{save_directory}{db_name}')
```

We then begin to loop through each of the files names. The first operation on each file is the determination of the proper file extension.
```python
  for file in file_list:
        # set up a variable ending the file name with each extension version
        url1 = 'https://s3.amazonaws.com/tripdata/' + file + '.zip'
        url2 = 'https://s3.amazonaws.com/tripdata/' + file + '.csv.zip'
        
        # test the validity of the file extensions
        response = requests.get(url1)
        if response.status_code != 200:
            response = requests.get(url2)
            if response.status_code != 200:
                print(f'{file} is unavailable')
                continue
```
This tests checks the newest version of the extension first, then the original syntax, and skips to the next file name in the list if the file is not found at all.

Next we need to access the file contents. We use io.BytesIO to create an in memory object that can be operated on as if it were a file. We extract the file contents with ZipFile, selecting just the .csv file for the following step.

```python
# io.BytesIO transforms response.content into a 'file' in memory
        with zipfile.ZipFile(io.BytesIO(response.content)) as zip_contents:
            file_list = zip_contents.namelist()
            with zip_contents.open(file_list[0]) as tempfile:
```

Over the eight years spanning 2013-2021 naming conventions and date formats for the data changed multiple times. To get a single database we can query from, we need to manipulate the data with Pandas into a single standardized format.
```Python
      #load the data into a Pandas DataFrame for transformation operations
                bike_df = pd.read_csv(tempfile)
                # standardize the names of column headers
                bike_df = bike_df.rename(columns=header_format)
                # standardize the datetime format
                #  Year-Month-Day Hour:Minute:Second 
                bike_df['started_at'] = (pd.to_datetime(bike_df['started_at'])
                                         .dt.strftime('%Y-%m-%d %H:%M:%S'))
                bike_df['ended_at'] = (pd.to_datetime(bike_df['ended_at'])
                                       .dt.strftime('%Y-%m-%d %H:%M:%S'))
                # standardize naming conventions for members and non members
                bike_df['member_casual'] = (bike_df['member_casual']
                .replace({'Subscriber':'member','Customer':'casual'
                }))
```
Now that we have clean data in a standarized format, the file can be appended to a SQL database and written to the disk. We will use Pandas and the SQLAlchemy connection, 'engine', to make this happen.

```Python
 # preparate for the index to be dropped
                bike_df.sort_values(by=['started_at'], ignore_index=True)
                # insert the file contents into a sqlite database    
                bike_df.to_sql('bikedata', con = engine, if_exists='append', index=False)
```

All that is left is to close our 'files' and free up memory for the next loop iteration.

```Python
       # clean up and free memory for the next file
            tempfile.close()
        zip_contents.close()
        response.close()
        del url1, url2, response 
        del file_list, tempfile, bike_df, zip_contents
        print(f'Successfully extracted and cleaned {file}')
```

## Calculating Lifespan
Now that we have a record of every trip that has been taken, we can find the first time and the last time that each bike riden. 

While we are at it, we will also count the total number of times each bike has been ridden, and the total number of seconds each bike has been ridden for. The SQLAlchemy connection will be used by Pandas to perform the SQL query and store the results in a DataFrame.

```Python
# the WHERE statement here is used to remove rows where the bike_id is not tracked
bikeinfo_df = pd.read_sql("""
SELECT 
    ride_id AS "Bike ID",
    MIN(started_at) AS "First Ride",
    MAX(started_at) AS "Last Ride",
    SUM(ROUND((JULIANDAY(ended_at) - JULIANDAY(started_at)) * 86400))
    AS "Total Ride Time (Seconds)",
    COUNT(started_at) AS "Total Rides"
FROM bikedata
WHERE strftime('%Y-%m', started_at) < '2021-02'
GROUP BY ride_id;
    """,con = engine)
```
JULIANDAY is used to convert the datetimes into a form that can be operated on mathematically. This is used to find the ride time of each trip.
-  The default value is a decimal number in days, so we multiply by 86400 to convert to seconds, as our level of measured precision is to the second.
- SUM is used to find the total ride time for all trips.

The WHERE statement is used to exclude new bikes added in February of 2021 from the analysis.

GROUPBY is used to specify that aggregate values are to be calculated by ride_id.

To get lifespan, we create a new DataFrame column with the difference in time between the last rast and the first ride for each bike.

```Python
bikeinfo_df['Life Span (Years)'] = (
    pd.to_datetime(bikeinfo_df['Last Ride']).dt.year
    - pd.to_datetime(bikeinfo_df['First Ride']).dt.year)
```
Note: As of February 2021 there were only 14,000 bikes in service, and more than 1 million trips taken for the month. That is an average of two rides a day for each bike.  Because of this, the analysis will assume that any bike without a record of being riden for a given month is no longer in service.
## Analysis

Since Citibike began in 2013, more than 33,000 unique bike IDs have been observed in the publc data. As of January 2021, when Citibike stopped sharing information on bike ID, only 14,000 of those bikes were still in use.

[<img src="https://github.com/bakerv/citibike-data-extraction_2013-2021/blob/main/images/overall_fleet_status.PNG">](https://public.tableau.com/app/profile/victor.baker/viz/NYCCitibikeBicycleLongevityAnalysis2013-2021/NYCCitibikeBicycleLongevityAnalysis)

Diving a little deeper in the data, we see that varies greatly depending on which year a given bike was initially put in service.

 The initial cohort, released in 2013, had an average lifespan of 5.8 years and nearly 1800 riding hours. The majority of these 2013 bikes were still in service in 2020, when they were replaced by 8000 brand new bikes. In contrast to this sucess story is 2015, the worst performing cohort of bikes to date. These 2500 bikes barely lasted a year, and only saw an average of 600 riding hours. 

 The most recent cohort we can draw conclusions from is 2016. The majority of these bikes have been removed from service, with a large number retired in Q4 2020. This cohort has an average of lifespan of 3.5 years. 
 
 [Check out the dashboard](https://public.tableau.com/app/profile/victor.baker/viz/NYCCitibikeBicycleLongevityAnalysis2013-2021/NYCCitibikeBicycleLongevityAnalysis) to dive further into the cohort statistics. 

 What does this mean for the current phase 3 expansion?
-  A signifigant amount of normal attrition will occur during the four year expansion process.

- If we expect bikes to have a lifespan of four years, then 32,000 additional bikes will need to be added after January 2021 to reach the 40,000 bike goal. Roughly 10,000 per year.

- After 2023, 10,000 bikes will need to be added to the fleet annually to maintain availability at 40,000 bikes.

- The current expansion rate will become the required maintenance rate. It is quite possible this was planned.

## Limitations

There is no way for us to know why bikes have been removed from service. Citibike does not release information on bike maintenance activities, and removed tracking data for individual bikes in February 2021.

We do not know if bike ID changed at any point for a given bike. Were those 2500 short-lived bikes from 2015 refurbished and released with new IDs? Did they get scrapped for parts for that long lived 2013 cohort?





