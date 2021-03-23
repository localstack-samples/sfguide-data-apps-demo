# Overview

This project provides a Snowflake demo focused on Data Application workloads.

The demo runs a single-page web application (powered by node.js) showcasing a few charts.  The data for the charts flows from Snowflake, using the Citibike data.  The web server backend connects to Snowflake using the node.js Snowflake SDK.

In addition to the website, this repository contains load testing scripts to showcase Snowflake's ability to handle concurrency, often a key concern among data application builders.  One load testing script showcases our caching ability (the result cache specifically), and the other showcases multi-clustering.

There are instructions below for setting up the application and running the demo.

# Installation

## Prerequisites

This demo assumes you have set up the citibike v3 demo (set to **normal** size).  In particular, this demo requires that you have already run the **load trips** and **get weather** core vignette SQL scripts.  

## Installing Citibike Web Application
- Download [node.js](https://nodejs.org/en/download/)
- Clone this repository
- Install the Citibike web application package dependencies
  - Navigate inside the top-level directory of this project and run `npm install`
- Install load testing toolkit
  - Run `npm install -g artillery` (if you get an access error, I suggest not using `sudo` but instead changing ownership of the `/usr/local/lib/node_modules/` directory and all subdirectories like so: `chown -R <user> /usr/local/lib/node_modules/`. )

## Snowflake Demo Account Configuration

The application backend connects to a Snowflake instance when the server is started. For the application to properly make that connection, you must specify how to connect to your Snowflake demo account.
- Create a `config.js` file in the top-level project directory from the template configuration file (`config-template.js`)
  - For example, `cp config-template.js config.js`
- Edit the `config.js` with your demo instance details
  - The snowflake user you configure must have select privileges on the citibike tables (specifically the citibike.demo.trips table).  The snowflake user must also be able to create the weather table in the citibike.demo schema and create a warehouse, which is done in the next section.  For simplicity, I tend to use `john`, the user from the core citibike demo; however, in production you'd wouldn't want to do this!  A future extension here could be to use a service user for the application.

## Snowflake Warehouse for Citibike Website Workloads

This demo runs all Snowflake queries in a single warehouse. To create that warehouse:
- Run `node scripts/setup/create_web_wh.js`

To leverage the result cache for the weather-related query, this application has to use a table instead of the trips_weather_vw.  The script below creates the appropriate table:
- Run `node scripts/setup/create_weather_table.js`

# Running The Demo

## Start Citibike Web Application

To access the website, you must first start the web server.
- From within the top-level project directory, run `npm start`

## View the Website

After the http server is running, you can access the website from your browser.
- Navigate to http://localhost:3000

## Simulate traffic

While it's easy to show the website will load quickly for a single user through the browser, we also want to showcase that Snowflake can handle concurrency well.  The following scripts simulate web traffic by making api calls. Run of the scripts follow this pattern:   
- `artillery run scripts/load_tests/<testname>` (replace _testname_ with name of test)

Current tests include:
- `all.yaml`: hit all 3 api endpoints (good for demoing result cache)
  - At peak of the load test, we simulating 100 users hitting the website every second for 10 seconds.
- `montly_random.yaml`: hit monthly trip counts api endpoint with random dates (good for demoing multi-clustering)
  - As this load test runs, the small warehouse will scale nicely over a little more than 2 minutes from 1 cluster to 4 clusters
  - If you turn off multi-clustering, queuing will build up
  - The random date range for this load test is limited to 1 year (~ 20M trips in date range).

# Miscellaneous Details

- The created warehouse is set to a small, with multi-clustering between 1 and 5.  Feel free to increase the warehouse size around if you'd like to showcase snappier performance.  I find after the cache warms, small works fine and helps limit costs for SF! 



MATERIALIZED EXTENSION:

CREATE OR REPLACE MATERIALIZED VIEW COUNT_BY_DAY_MVW AS
select COUNT(*) as trip_count, date_trunc('DAY', starttime) as starttime from demo.trips group by date_trunc('DAY', starttime);

CREATE OR REPLACE MATERIALIZED VIEW COUNT_BY_MONTH_MVW AS
select COUNT(*) as trip_count, MONTHNAME(starttime) as month, MIN(starttime) as starttime from demo.trips group by month;

BEFORE MATERIALIZED (MAX 5)
  Response time (msec):
    min: 166.3
    max: 7477.1
    median: 544.4
    p95: 1645.6
    p99: 3469.1

    var mquery = 'select SUM(trip_count) as trip_count, MONTHNAME(starttime) as month ' +
            'from demo.COUNT_BY_DAY_MVW ' +
            'where starttime between ? and ? ' +
            'group by MONTH(starttime), MONTHNAME(starttime) ' +
            'order by MONTH(starttime);'


    var mquery = 'select * from demo.COUNT_BY_MONTH_MVW order by starttime;'

MATERIALIZED (MAX 3)
  Response time (msec):
    min: 172.8
    max: 2333.3
    median: 287.1
    p95: 703.8
    p99: 1340.2
