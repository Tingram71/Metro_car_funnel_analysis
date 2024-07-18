# Metrocar Rideshare App Funnel Analysis Project
## Overview:
This repository houses the final assessment project for the Data Analysis Program at MasterSchool. The project focused on analyzing the customer funnel of Metrocar, a ride-sharing app comparable to Uber and Lyft, utilizing SQL for data querying and Tableau for data visualization.
## Motivation:
The project concentrated on performing an in-depth funnel analysis to pinpoint areas for enhancement in Metrocar's customer journey. Its goal was to answer specific business questions, offering insights for optimization and data-driven recommendations.
## Project:
The project's objective was to perform a funnel analysis on Metrocar's customer journey to identify drop-off points and recommend improvement strategies. Funnel analysis reveals where users exit the funnel, helping to improve desired outcomes such as sales and conversions.
## Metrocar's Funnel:
Metrocar operates as a ride-sharing intermediary, linking riders with drivers via a user-friendly mobile application. The customer funnel encompasses stages such as app download, signup, ride request, driver acceptance, ride completion, payment, and review. Drop-offs at each stage are analyzed to identify opportunities for optimization.
## Ad-Hoc Analysis:

#### Total app downloads:
How many times has the Metrocar app been downloaded?
```sql
SELECT count(*) AS total_downloads
FROM app_downloads;
````
| total_downloads |
| --------------- |
| 23608           |

#### Total users signed up on the app:
How many users who have signed up on the app?
```sql
SELECT COUNT(user_id) AS total_signups
FROM signups;
````
| total_signups |
| ------------- |
| 17623         |

#### Total rides requested through the app:
How many rides have been requested through the app?
```SQL
SELECT COUNT(request_ts) AS total_ride_requests
FROM ride_requests;
````
| total_ride_requests |
| ------------------- |
| 385477              |

#### Total rides requested and completed through the app:
How many rides have been requested and completed through the app?
````sql
SELECT
  COUNT(request_ts) AS total_ride_requests,
  COUNT(dropoff_ts) AS total_completed_rides
FROM
  ride_requests;
`````
| total_ride_requests | total_completed_rides |
| ------------------- | --------------------- |
| 385477              | 223652                |

#### Total rides requested and count of unique users requesting a ride:
```sql
SELECT
  COUNT(request_ts) AS total_rides_requested,
  COUNT(DISTINCT user_id) AS total_unique_users
FROM
  ride_requests;
````
| total_rides_requested | total_unique_users |
| --------------------- | ------------------ |
| 385477                | 12406              |

#### Average duration of a ride from pick up to drop off:
```sql
SELECT
  AVG(dropoff_ts - pickup_ts) AS avg_ride_duration
FROM
  ride_requests;
````
| avg_ride_duration |
| ----------------- |
| 00:52:36.738773   |

#### Count of rides accepted by a driver:
````sql
SELECT
  COUNT(accept_ts) AS total_rides_driver_accepted
FROM
  ride_requests;
````
| total_rides_driver_accepted |
| --------------------------- |
| 248379                      |

#### Total rides that successfully collected payments and total collected
````sql
SELECT
  COUNT(transaction_id) AS total_transactions,
  SUM(purchase_amount_usd) AS total_revenue
FROM
  transactions
WHERE
  charge_status = 'Approved';
````
| transaction_count | total_revenue     |
| ----------------- | ----------------- |
| 212628            | 4251667.609999995 |

#### Total rides requested on each platform:
````sql
SELECT
  platform,
  COUNT(r.request_ts) AS total_ride_requests
FROM
  app_downloads a
  LEFT JOIN signups s ON a.app_download_key = s.session_id
  LEFT JOIN ride_requests r ON s.user_id = r.user_id
GROUP BY
  platform
ORDER BY
  total_ride_requests DESC;
````
| platform | total_ride_requests |
| -------- | ------------------- |
| ios      | 234693              |
| android  | 112317              |
| web      | 38467               |

#### What is the drop-off from users signing up to users requesting a ride?
```sql
SELECT
  COUNT(distinct s.user_id) AS total_signups,
  COUNT(distinct r.user_id) AS total_ride_requests,
  COUNT(distinct s.user_id) - COUNT(distinct r.user_id) AS total_droppoffs,
  1 - (
    COUNT(distinct r.user_id) / COUNT(distinct s.user_id)::numeric
  ) AS dropoff_rate
FROM
  signups s
  LEFT JOIN ride_requests r USING (user_id);
````
| total_signups | total_ride_requests | total_droppoffs | dropoff_rate           |
| ------------- | ------------------- | --------------- | ---------------------- |
| 17623         | 12406               | 5217            | 0.29603359246439312262 |

29.6%

#### Of the users that signed up on the app, what percentage these users completed a ride?
````sql
SELECT
  COUNT(DISTINCT s.user_id) AS total_signups,
  COUNT(DISTINCT r.user_id) AS total_rides_completed,
  COUNT(DISTINCT r.user_id) / COUNT(DISTINCT s.user_id)::FLOAT * 100 AS percentage_completed
FROM
  signups s
  LEFT JOIN ride_requests r ON s.user_id = r.user_id
  AND r.dropoff_ts IS NOT NULL;
````
| total_signups | total_rides_completed | percentage_completed |
| ------------- | --------------------- | -------------------- |
| 17623         | 6233                  | 35.3685524598536     |


####  Using the percent of previous approach, what are the user-level conversion rates for the first 3 stages of the funnel (app download to signup and signup to ride requested)?
````sql
WITH
  
  totals AS (
    SELECT
      COUNT(DISTINCT a.app_download_key) AS total_app_downloads,
      COUNT(DISTINCT s.user_id) AS total_users_signed_up,
      COUNT(DISTINCT r.user_id) AS total_users_ride_requested
    FROM
      app_downloads a
      LEFT JOIN signups s ON a.app_download_key = s.session_id
      LEFT JOIN ride_requests r ON s.user_id = r.user_id
  ),
  funnel_stages AS (
    SELECT
      0 AS funnel_step,
      'downloads' AS funnel_name,
      total_app_downloads AS value
    FROM
      totals
    UNION
    SELECT
      1 AS funnel_step,
      'signups' AS funnel_name,
      total_users_signed_up AS value
    FROM
      totals
    UNION
    SELECT
      2 AS funnel_step,
      'ride_requested' AS funnel_name,
      total_users_ride_requested AS value
    FROM
      totals
  )
SELECT
  *,
  value::float / LAG(value) OVER ( 
    ORDER BY
      funnel_step
  ) AS previous_value
FROM
  funnel_stages
ORDER BY
  funnel_step;
````
| funnel_step | funnel_name    | value | previous_value     |
| ----------- | -------------- | ----- | ------------------ |
| 0           | downloads      | 23608 |                    |
| 1           | signups        | 17623 | 0.7464842426296171 |
| 2           | ride_requested | 12406 | 0.7039664075356069 |

####  Using the percent of top approach, what are the user-level conversion rates for the first 3 stages of the funnel (app download to signup and signup to ride requested)?
````sql
WITH
  
  totals AS (
    SELECT
      COUNT(DISTINCT a.app_download_key) AS total_app_downloads,
      COUNT(DISTINCT s.user_id) AS total_users_signed_up,
      COUNT(DISTINCT r.user_id) AS total_users_ride_requested
    FROM
      app_downloads a
      LEFT JOIN signups s ON a.app_download_key = s.session_id
      LEFT JOIN ride_requests r ON s.user_id = r.user_id
  ),
  funnel_stages AS (
    SELECT
      0 AS funnel_step,
      'downloads' AS funnel_name,
      total_app_downloads AS value
    FROM
      totals
    UNION
    SELECT
      1 AS funnel_step,
      'signups' AS funnel_name,
      total_users_signed_up AS value
    FROM
      totals
    UNION
    SELECT
      2 AS funnel_step,
      'ride_requested' AS funnel_name,
      total_users_ride_requested AS value
    FROM
      totals
  )
SELECT
  *,
  value::float / FIRST_VALUE(value) OVER (
    ORDER BY
      funnel_step
  ) AS top_value
FROM
  funnel_stages
ORDER BY
  funnel_step;
````
| funnel_step | funnel_name    | value | top_value          |
| ----------- | -------------- | ----- | ------------------ |
| 0           | downloads      | 23608 | 1                  |
| 1           | signups        | 17623 | 0.7464842426296171 |
| 2           | ride_requested | 12406 | 0.5254998305659099 |

#### Using the percent of previous approach, what are the user-level conversion rates for the following 3 stages of the funnel? 1. signup, 2. ride requested, 3. ride completed
````sql
WITH
  totals AS (
    SELECT
      COUNT(DISTINCT a.app_download_key) AS total_app_downloads,
      COUNT(DISTINCT s.user_id) AS total_users_signed_up,
      COUNT(DISTINCT r.user_id) AS total_users_ride_requested,
      COUNT(
        DISTINCT CASE
          WHEN r.dropoff_ts IS NOT NULL THEN r.user_id
        END
      ) AS total_users_ride_completed
    FROM
      app_downloads a
      LEFT JOIN signups s ON a.app_download_key = s.session_id
      LEFT JOIN ride_requests r ON s.user_id = r.user_id
  ),
  funnel_stages AS (
    SELECT
      0 AS funnel_step,
      'downloads' AS funnel_name,
      total_app_downloads AS value
    FROM
      totals
    UNION
    SELECT
      1 AS funnel_step,
      'signups' AS funnel_name,
      total_users_signed_up AS value
    FROM
      totals
    UNION
    SELECT
      2 AS funnel_step,
      'ride_requested' AS funnel_name,
      total_users_ride_requested AS value
    FROM
      totals
    UNION
    SELECT
      3 AS funnel_step,
      'ride_completed' AS funnel_name,
      total_users_ride_completed AS value
    FROM
      totals
      --WHERE dropoff_ts IS NOT NULL
  )
SELECT
  *,
  value::float / LAG(value) OVER (
    ORDER BY
      funnel_step
  ) AS previous_value
FROM
  funnel_stages
ORDER BY
  funnel_step;
````
| funnel_step | funnel_name    | value | previous_value     |
| ----------- | -------------- | ----- | ------------------ |
| 0           | downloads      | 23608 |                    |
| 1           | signups        | 17623 | 0.7464842426296171 |
| 2           | ride_requested | 12406 | 0.7039664075356069 |
| 3           | ride_completed | 6233  | 0.5024181847493149 |


 #### Using the percent of top approach, what are the  user-level conversion rates for the following 3 stages of the funnel? 1. signup, 2. ride requested, 3. ride completed 
````sql
WITH
  totals AS (
    SELECT
      COUNT(DISTINCT a.app_download_key) AS total_app_downloads,
      COUNT(DISTINCT s.user_id) AS total_users_signed_up,
      COUNT(DISTINCT r.user_id) AS total_users_ride_requested,
      COUNT(
        DISTINCT CASE
          WHEN r.dropoff_ts IS NOT NULL THEN r.user_id
        END
      ) AS total_users_ride_completed
    FROM
      app_downloads a
      LEFT JOIN signups s ON a.app_download_key = s.session_id
      LEFT JOIN ride_requests r ON s.user_id = r.user_id
  ),
  funnel_stages AS (
   
    SELECT
      1 AS funnel_step,
      'signups' AS funnel_name,
      total_users_signed_up AS value
    FROM
      totals
    UNION
    SELECT
      2 AS funnel_step,
      'ride_requested' AS funnel_name,
      total_users_ride_requested AS value
    FROM
      totals
    UNION
    SELECT
      3 AS funnel_step,
      'ride_completed' AS funnel_name,
      total_users_ride_completed AS value
    FROM
      totals
      
  )
SELECT
  *,
  value::float / FIRST_VALUE(value) OVER (
    ORDER BY
      funnel_step
  ) AS top_value
FROM
  funnel_stages
ORDER BY
  funnel_step;
````
| funnel_step | funnel_name    | value | top_value          |
| ----------- | -------------- | ----- | ------------------ |
| 1           | signups        | 17623 | 1                  |
| 2           | ride_requested | 12406 | 0.7039664075356069 |
| 3           | ride_completed | 6233  | 0.353685524598536  |

#### Funnel Percent of Top: Calulates total users for each funnel step with percentage of users who completed that step from the initial total users who siged up on the app.
````sql
WITH
  totals AS (
    SELECT
      COUNT(DISTINCT a.app_download_key) AS total_app_downloads,
      COUNT(DISTINCT s.user_id) AS total_users_signed_up,
      COUNT(DISTINCT r.user_id) AS total_users_ride_requested,
      COUNT(
        DISTINCT CASE
          WHEN r.accept_ts IS NOT NULL THEN r.user_id
        END
      ) AS total_users_ride_accepted,
      COUNT(
        DISTINCT CASE
          WHEN r.dropoff_ts IS NOT NULL THEN r.user_id
        END
      ) AS total_users_ride_completed,

      COUNT(
        DISTINCT CASE
          WHEN t.charge_status = 'Approved' THEN r.user_id
        END
      ) AS total_users_transaction_completed,
      COUNT(DISTINCT rev.user_id) AS total_users_review_completed
    FROM
      app_downloads a
      LEFT JOIN signups s ON a.app_download_key = s.session_id
      LEFT JOIN ride_requests r ON s.user_id = r.user_id
      LEFT JOIN transactions t ON r.ride_id = t.ride_id
      LEFT JOIN reviews rev ON r.user_id = rev.user_id
  ),
  funnel_stages AS (
    SELECT
      0 AS funnel_step,
      'downloads' AS funnel_name,
      total_app_downloads AS value
    FROM
      totals
    UNION
    SELECT
      1 AS funnel_step,
      'signups' AS funnel_name,
      total_users_signed_up AS value
    FROM
      totals
    UNION
    SELECT
      2 AS funnel_step,
      'ride_requested' AS funnel_name,
      total_users_ride_requested AS value
    FROM
      totals
    UNION
    SELECT
      3 AS funnel_step,
      'ride_accepted' AS funnel_name,
      total_users_ride_accepted AS value
    FROM
      totals
    UNION
    SELECT
      4 AS funnel_step,
      'ride_completed' AS funnel_name,
      total_users_ride_completed AS value
    FROM
      totals
    UNION
    SELECT
      5 AS funnel_step,
      'payment_completed' AS funnel_name,
      total_users_transaction_completed AS value
    FROM
      totals
    UNION
    SELECT
      6 AS funnel_step,
      'review_completed' AS funnel_name,
      total_users_review_completed AS value
    FROM
      totals
  )
SELECT
  *,
  value::float / LAG(value) OVER (
    ORDER BY
      funnel_step
  ) AS previous_value, 
  value::float / FIRST_VALUE(value) OVER (
    ORDER BY
      funnel_step
  ) AS top_value
FROM
  funnel_stages
ORDER BY
  funnel_step;
````
| funnel_step | funnel_name       | value | top_value           | previous_value     |
| ----------- | ----------------- | ----- | ------------------- | ------------------ |
| 0           | downloads         | 23608 | 1                   |                    |
| 1           | signups           | 17623 | 0.7464842426296171  | 0.7464842426296171 |
| 2           | ride_requested    | 12406 | 0.5254998305659099  | 0.7039664075356069 |
| 3           | ride_accepted     | 12278 | 0.5200779396814639  | 0.9896824117362566 |
| 4           | ride_completed    | 6233  | 0.26402067095899695 | 0.5076559700276918 |
| 5           | payment_completed | 6233  | 0.26402067095899695 | 1                  |
| 6           | review_completed  | 4348  | 0.18417485598102337 | 0.6975774105567143 |

#### To pull aggregated data for each funnel step by platform, user age range and download date of app:

````sql
WITH
  user_details AS (
    SELECT
      app_download_key,
      user_id,
      platform,
      age_range,
      date(download_ts) AS download_dt
    FROM
      app_downloads a
      LEFT JOIN signups s ON a.app_download_key = s.session_id
  ),
  downloads AS (
    SELECT
      0 AS step,
      'download' AS name,
      platform,
      age_range,
      download_dt,
      COUNT(DISTINCT app_download_key) AS users_count,
      0 AS count_rides
    FROM
      user_details
    GROUP BY
      platform,
      age_range,
      download_dt
  ),
  signup AS (
    SELECT
      1 AS step,
      'signup' AS name,
      user_details.platform,
      user_details.age_range,
      user_details.download_dt,
      COUNT(DISTINCT user_id) AS users_count,
      0 AS count_rides
    FROM
      signups
      JOIN user_details USING (user_id)
    WHERE
      signup_ts IS NOT NULL
    GROUP BY
      3,
      4,
      5
  ),
  requested AS (
    SELECT
      2 AS step,
      'ride_requested' AS name,
      user_details.platform,
      user_details.age_range,
      user_details.download_dt,
      COUNT(DISTINCT user_id) AS users_count,
      COUNT(DISTINCT ride_id) AS count_rides
    FROM
      ride_requests
      JOIN user_details USING (user_id)
    where
      request_ts IS NOT NULL
    GROUP BY
      user_details.platform,
      user_details.age_range,
      user_details.download_dt
  ),
  ride_accepted AS (
    SELECT
      3 AS step,
      'driver_accepted' AS name,
      user_details.platform,
      user_details.age_range,
      user_details.download_dt,
      COUNT(DISTINCT user_id) AS users_count,
      COUNT(DISTINCT ride_id) AS count_rides
    FROM
      ride_requests
      JOIN user_details USING (user_id)
    WHERE
      accept_ts IS NOT NULL
    GROUP BY
      user_details.platform,
      user_details.age_range,
      user_details.download_dt
  ),
  ride_completed AS (
    SELECT
      4 AS step,
      'ride_completed' AS name,
      user_details.platform,
      user_details.age_range,
      user_details.download_dt,
      COUNT(DISTINCT user_id) AS users_count,
      COUNT(DISTINCT ride_id) AS count_rides
    FROM
      ride_requests
      JOIN user_details USING (user_id)
    WHERE
      dropoff_ts IS NOT NULL
    GROUP BY
      user_details.platform,
      user_details.age_range,
      user_details.download_dt
  ),
  payment AS (
    SELECT
      5 AS step,
      'payment_completed' AS name,
      user_details.platform,
      user_details.age_range,
      user_details.download_dt,
      COUNT(DISTINCT user_id) AS users_count,
      COUNT(DISTINCT t.ride_id) AS count_rides
    FROM
      transactions t
      LEFT JOIN ride_requests r ON t.ride_id = r.ride_id
      JOIN user_details using (user_id)
    WHERE
      charge_status = 'Approved'
    GROUP BY
      user_details.platform,
      user_details.age_range,
      user_details.download_dt
  ),
  review AS (
    SELECT
      6 AS step,
      'review' AS NAME,
      user_details.platform,
      user_details.age_range,
      user_details.download_dt,
      COUNT(DISTINCT user_id) AS users_count,
      COUNT(DISTINCT ride_id) AS count_rides
    FROM
      reviews
      JOIN user_details USING (user_id)
    GROUP BY
      user_details.platform,
      user_details.age_range,
      user_details.download_dt
  )
SELECT
  *
FROM
  downloads
UNION
SELECT
  *
FROM
  signup
UNION
SELECT
  *
FROM
  requested
UNION
SELECT
  *
FROM
  ride_accepted
UNION
SELECT
  *
FROM
  ride_completed
UNION
SELECT
  *
FROM
  payment
UNION
SELECT
  *
FROM
  review
ORDER BY
  step;
````
#### Calculations for total users, percent of previous step and percent of total users grouped by platform.

````sql
WITH
  totals AS (
    SELECT
      COUNT(DISTINCT a.app_download_key) AS total_app_downloads,
      COUNT(DISTINCT s.user_id) AS total_users_signed_up,
      COUNT(DISTINCT r.user_id) AS total_users_ride_requested,
      COUNT(
        DISTINCT CASE
          WHEN r.accept_ts IS NOT NULL THEN r.user_id
        END
      ) AS total_users_ride_accepted,
      COUNT(
        DISTINCT CASE
          WHEN r.dropoff_ts IS NOT NULL THEN r.user_id
        END
      ) AS total_users_ride_completed,

      COUNT(
        DISTINCT CASE
          WHEN t.charge_status = 'Approved' THEN r.user_id
        END
      ) AS total_users_transaction_completed,
      COUNT(DISTINCT rev.user_id) AS total_users_review_completed, platform
    FROM
      app_downloads a
      LEFT JOIN signups s ON a.app_download_key = s.session_id
      LEFT JOIN ride_requests r ON s.user_id = r.user_id
      LEFT JOIN transactions t ON r.ride_id = t.ride_id
      LEFT JOIN reviews rev ON r.user_id = rev.user_id
    GROUP BY platform
  ),
  funnel_stages AS (
    SELECT
      0 AS funnel_step,
      'downloads' AS funnel_name, platform,
      total_app_downloads AS value
    FROM
      totals
    UNION
    SELECT
      1 AS funnel_step,
      'signups' AS funnel_name, platform,
      total_users_signed_up AS value 
    FROM
      totals
    UNION
    SELECT
      2 AS funnel_step,
      'ride_requested' AS funnel_name, platform,
      total_users_ride_requested AS value
    FROM
      totals
    UNION
    SELECT
      3 AS funnel_step, platform,
      'ride_accepted' AS funnel_name,
      total_users_ride_accepted AS value
    FROM
      totals
    UNION
    SELECT
      4 AS funnel_step,
      'ride_completed' AS funnel_name, platform,
      total_users_ride_completed AS value
    FROM
      totals
    UNION
    SELECT
      5 AS funnel_step,
      'payment_completed' AS funnel_name, platform,
      total_users_transaction_completed AS value
    FROM
      totals
    UNION
    SELECT
      6 AS funnel_step,
      'review_completed' AS funnel_name, platform,
      total_users_review_completed AS value
    FROM
      totals
  )
SELECT
  *,
  value::float / LAG(value) OVER (
    ORDER BY
      funnel_step
  ) AS previous_value, 
  value::float / FIRST_VALUE(value) OVER (
    ORDER BY
      funnel_step
  ) AS top_value
FROM
  funnel_stages
ORDER BY
  funnel_step;
````
#### Count of users and rides for 'ride request' step of the funnel grouped by hour of request, month and day:

````sql
WITH
  user_details AS (
    SELECT
      app_download_key,
      user_id, EXTRACT(day from request_ts) AS request_day,
      EXTRACT(
        hour
        from
          request_ts
      ) AS request_hours,
      EXTRACT(
        month
        from
          request_ts
      ) AS request_month
    FROM
      app_downloads a
      LEFT JOIN signups s ON a.app_download_key = s.session_id
      LEFT JOIN ride_requests r USING (user_id)
  ),
  requested AS (
    SELECT
      2 AS step,
      'ride_requested' AS name,
      user_details.request_hours,
      user_details.request_month,
    user_details.request_day,
      COUNT(DISTINCT user_id) AS users_count,
      COUNT(DISTINCT ride_id) AS count_rides
    FROM
      ride_requests
      JOIN user_details USING (user_id)
    where
      request_ts IS NOT NULL
    GROUP BY
      user_details.request_hours,
      user_details.request_month,
    user_details.request_day
  )
SELECT
  *
FROM
  requested
ORDER BY
  step;
````
#### Count of users and rides for each funnel step grouped by platform, age range and trip duration:

````sql
WITH
  user_details AS (
    SELECT
      app_download_key,
      s.user_id,
      platform,
      age_range,
      dropoff_ts - pickup_ts AS trip_duration
    FROM
      app_downloads a
      LEFT JOIN signups s ON a.app_download_key = s.session_id
    LEFT JOIN ride_requests r ON s.user_id = r.user_id
  ),
  downloads AS (
    SELECT
      0 AS step,
      'download' AS name,
      platform,
      age_range,
      trip_duration,
      COUNT(DISTINCT app_download_key) AS users_count,
      0 AS count_rides
    FROM
      user_details
    GROUP BY
      platform,
      age_range,
      trip_duration
  ),
  signup AS (
    SELECT
      1 AS step,
      'signup' AS name,
      user_details.platform,
      user_details.age_range,
      user_details.trip_duration,
      COUNT(DISTINCT user_id) AS users_count,
      0 AS count_rides
    FROM
      signups
      JOIN user_details USING (user_id)
    WHERE
      signup_ts IS NOT NULL
    GROUP BY
      3,
      4,
      5
  ),
  requested AS (
    SELECT
      2 AS step,
      'ride_requested' AS name,
      user_details.platform,
      user_details.age_range,
      user_details.trip_duration,
      COUNT(DISTINCT user_id) AS users_count,
      COUNT(DISTINCT ride_id) AS count_rides
    FROM
      ride_requests
      JOIN user_details USING (user_id)
    where
      request_ts IS NOT NULL
    GROUP BY
      user_details.platform,
      user_details.age_range,
      user_details.trip_duration
  ),
  ride_accepted AS (
    SELECT
      3 AS step,
      'driver_accepted' AS name,
      user_details.platform,
      user_details.age_range,
      user_details.trip_duration,
      COUNT(DISTINCT user_id) AS users_count,
      COUNT(DISTINCT ride_id) AS count_rides
    FROM
      ride_requests
      JOIN user_details USING (user_id)
    WHERE
      accept_ts IS NOT NULL
    GROUP BY
      user_details.platform,
      user_details.age_range,
      user_details.trip_duration
  ),
  ride_completed AS (
    SELECT
      4 AS step,
      'ride_completed' AS name,
      user_details.platform,
      user_details.age_range,
      user_details.trip_duration,
      COUNT(DISTINCT user_id) AS users_count,
      COUNT(DISTINCT ride_id) AS count_rides
    FROM
      ride_requests
      JOIN user_details USING (user_id)
    WHERE
      dropoff_ts IS NOT NULL
    GROUP BY
      user_details.platform,
      user_details.age_range,
      user_details.trip_duration
  ),
  payment AS (
    SELECT
      5 AS step,
      'payment_completed' AS name,
      user_details.platform,
      user_details.age_range,
      user_details.trip_duration,
      COUNT(DISTINCT user_id) AS users_count,
      COUNT(DISTINCT t.ride_id) AS count_rides
    FROM
      transactions t
      LEFT JOIN ride_requests r ON t.ride_id = r.ride_id
      JOIN user_details using (user_id)
    WHERE
      charge_status = 'Approved'
    GROUP BY
      user_details.platform,
      user_details.age_range,
      user_details.trip_duration
  ),
  review AS (
    SELECT
      6 AS step,
      'review' AS NAME,
      user_details.platform,
      user_details.age_range,
      user_details.trip_duration,
      COUNT(DISTINCT user_id) AS users_count,
      COUNT(DISTINCT ride_id) AS count_rides
    FROM
      reviews
      JOIN user_details USING (user_id)
    GROUP BY
      user_details.platform,
      user_details.age_range,
      user_details.trip_duration
  )
SELECT
  *
FROM
  downloads
UNION
SELECT
  *
FROM
  signup
UNION
SELECT
  *
FROM
  requested
UNION
SELECT
  *
FROM
  ride_accepted
UNION
SELECT
  *
FROM
  ride_completed
UNION
SELECT
  *
FROM
  payment
UNION
SELECT
  *
FROM
  review
ORDER BY
  step;
````
#### User count and ride count for each funnel step grouped by platform, age range, download date and trip fare

````sql
WITH
  user_details AS (
    SELECT
      app_download_key,
      s.user_id,
      platform,
      age_range,
      date(download_ts) AS download_dt,
    AVG(purchase_amount_usd) AS trip_fare
    FROM
      app_downloads a
      LEFT JOIN signups s ON a.app_download_key = s.session_id
    LEFT JOIN ride_requests r ON s.user_id = r.user_id
    LEFT JOIN transactions t ON r.ride_id = t.ride_id 
    WHERE purchase_amount_usd IS NOT NULL
    GROUP BY 1,2,3,4,5
  ),
  downloads AS (
    SELECT
      0 AS step,
      'download' AS name,
      platform,
      age_range,
      download_dt,trip_fare,
      COUNT(DISTINCT app_download_key) AS users_count,
      0 AS count_rides
    FROM
      user_details
    GROUP BY
      platform,
      age_range,
      download_dt,
   trip_fare
  ),
  signup AS (
    SELECT
      1 AS step,
      'signup' AS name,
      user_details.platform,
      user_details.age_range,
      user_details.download_dt,
   trip_fare,
      COUNT(DISTINCT user_id) AS users_count,
      0 AS count_rides
    FROM
      signups
      JOIN user_details USING (user_id)
    WHERE
      signup_ts IS NOT NULL
    GROUP BY
      3,
      4,
      5,
    trip_fare
  ),
  requested AS (
    SELECT
      2 AS step,
      'ride_requested' AS name,
      user_details.platform,
      user_details.age_range,
      user_details.download_dt,
    trip_fare,
      COUNT(DISTINCT user_id) AS users_count,
      COUNT(DISTINCT ride_id) AS count_rides
    FROM
      ride_requests
      JOIN user_details USING (user_id)
    where
      request_ts IS NOT NULL
    GROUP BY
      user_details.platform,
      user_details.age_range,
      user_details.download_dt,
    trip_fare
  ),
  ride_accepted AS (
    SELECT
      3 AS step,
      'driver_accepted' AS name,
      user_details.platform,
      user_details.age_range,
      user_details.download_dt,
      
    trip_fare,
    COUNT(DISTINCT user_id) AS users_count,
      COUNT(DISTINCT ride_id) AS count_rides
    FROM
      ride_requests
      JOIN user_details USING (user_id)
    WHERE
      accept_ts IS NOT NULL
    GROUP BY
      user_details.platform,
      user_details.age_range,
      user_details.download_dt,
    trip_fare
  ),
  ride_completed AS (
    SELECT
      4 AS step,
      'ride_completed' AS name,
      user_details.platform,
      user_details.age_range,
      user_details.download_dt,
   trip_fare,  
    COUNT(DISTINCT user_id) AS users_count,
      COUNT(DISTINCT ride_id) AS count_rides
    FROM
      ride_requests
      JOIN user_details USING (user_id)
    WHERE
      dropoff_ts IS NOT NULL
    GROUP BY
      user_details.platform,
      user_details.age_range,
      user_details.download_dt,
    trip_fare
  ),
  payment AS (
    SELECT
      5 AS step,
      'payment_completed' AS name,
      user_details.platform,
      user_details.age_range,
      user_details.download_dt,
    trip_fare, 
    COUNT(DISTINCT user_id) AS users_count,
      COUNT(DISTINCT t.ride_id) AS count_rides
    FROM
      transactions t
      LEFT JOIN ride_requests r ON t.ride_id = r.ride_id
      JOIN user_details using (user_id)
    WHERE
      charge_status = 'Approved'
    GROUP BY
      user_details.platform,
      user_details.age_range,
      user_details.download_dt,
    trip_fare
  ),
  review AS (
    SELECT
      6 AS step,
      'review' AS NAME,
      user_details.platform,
      user_details.age_range,
      user_details.download_dt,
   trip_fare,  
    COUNT(DISTINCT user_id) AS users_count,
      COUNT(DISTINCT ride_id) AS count_rides
    FROM
      reviews
      JOIN user_details USING (user_id)
    GROUP BY
      user_details.platform,
      user_details.age_range,
      user_details.download_dt,
    trip_fare
  )
SELECT
  *
FROM
  downloads
UNION
SELECT
  *
FROM
  signup
UNION
SELECT
  *
FROM
  requested
UNION
SELECT
  *
FROM
  ride_accepted
UNION
SELECT
  *
FROM
  ride_completed
UNION
SELECT
  *
FROM
  payment
UNION
SELECT
  *
FROM
  review
ORDER BY
  step;
````







