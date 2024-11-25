# Metrocar Rideshare App Funnel Analysis Project
## Overview:
This repository houses a Data Analysis project for customer funnel optimisation. The project focused on analyzing data from Metrocar, a ride-sharing app comparable to Uber and Lyft, utilizing SQL for data querying and Tableau for data visualization.
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

## Conclusions and Recommendations:

#### The funnel displayed in table form above represents user behavior and highlights areas of drop-off at each stage. Here are some insights:  

###   
1. **High signup conversion rate**  
   There is high download to signup conversion rate (74.7%). Users are engaging well after downloading.  

2. **Significant Drop-off**  
   From signups to ride requests, there's a 29.6% drop. This could indicate challenges in onboarding or a lack of incentive to request a ride.  

3. **Minimal Loss in Ride Fulfillment**  
   The drop-off from ride requested to ride accepted is very low (1%). Operational efficiency seems solid here.  

4. **Sharp Decline in Ride Completion**  
   Only 50.8% of users who requested rides complete them. This could be due to cancellations, availability issues, or changes in user decisions.  

5. **Payment Completion is Consistent**  
   No further loss from ride completion to payment completion. Users completing rides are reliable payers.  

6. **Low Review Engagement**  
   Only 69.7% of users who completed payments leave reviews. Engaging users to provide feedback is a challenge.  

### Suggestions  
1. **Focus on Signups to Ride Requests**  
   Simplify the process, offer incentives, or improve UX to reduce friction at this stage.  

2. **Analyze Ride Completion Drop-offs**  
   Study why users cancel or abandon rides (e.g., pricing, delays, availability). Address operational inefficiencies or offer ride guarantees.  

3. **Boost Review Completion**  
   Encourage reviews with incentives (e.g., discounts, points) or make it easier to leave feedback.  



#### The following queries are to pull aggregated data from the database for further analysis for example creating dashboards. The output is mostly too large to display. 
#### The first query each funnel step by platform, user age range and download date of app. Output too large to display (26901 rows).

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
| funnel_step | funnel_name       | platform      | value | previous_value      | top_value           |
| ----------- | ----------------- | ------------- | ----- | ------------------- | ------------------- |
| 0           | downloads         | web           | 2383  | null                | 1                   |
| 0           | downloads         | android       | 6935  | 2.9101972303818715  | 2.9101972303818715  |
| 0           | downloads         | ios           | 14290 | 2.06056236481615    | 5.996642887117079   |
| 1           | signups           | ios           | 10728 | 0.7507347795661301  | 4.501888375996643   |
| 1           | signups           | android       | 5148  | 0.4798657718120805  | 2.160302140159463   |
| 1           | signups           | web           | 1747  | 0.33935508935508935 | 0.7331095258078053  |
| 2           | ride_requested    | web           | 1237  | 0.7080709788208357  | 0.5190935795216114  |
| 2           | ride_requested    | android       | 3619  | 2.925626515763945   | 1.5186739404112464  |
| 2           | ride_requested    | ios           | 7550  | 2.0862116606797456  | 3.1682752832563996  |
| 3           | ios               | ride_accepted | 7471  | 0.9895364238410596  | 3.1351237935375575  |
| 3           | android           | ride_accepted | 3580  | 0.47918618658814083 | 1.502308015107008   |
| 3           | web               | ride_accepted | 1227  | 0.3427374301675978  | 0.5148971884179605  |
| 4           | ride_completed    | web           | 611   | 0.4979625101874491  | 0.2563994964330676  |
| 4           | ride_completed    | android       | 1830  | 2.9950900163666123  | 0.7679395719681075  |
| 4           | ride_completed    | ios           | 3792  | 2.0721311475409836  | 1.5912715065044063  |
| 5           | payment_completed | web           | 611   | 0.16112869198312235 | 0.2563994964330676  |
| 5           | payment_completed | android       | 1830  | 2.9950900163666123  | 0.7679395719681075  |
| 5           | payment_completed | ios           | 3792  | 2.0721311475409836  | 1.5912715065044063  |
| 6           | review_completed  | web           | 424   | 0.11181434599156118 | 0.17792698279479646 |
| 6           | review_completed  | ios           | 2651  | 6.252358490566038   | 1.112463281577843   |
| 6           | review_completed  | android       | 1273  | 0.48019615239532254 | 0.5342005874947545  |

````
#### Count of users and rides for 'ride request' step of the funnel grouped by hour of request, month and day. Output too large to display (8655 rows). 

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
#### Count of users and rides for each funnel step grouped by platform, age range and trip duration: Output too large to display (8043 rows). 

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
#### User count and ride count for each funnel step grouped by platform, age range, download date and trip fare. Output to large to dsiplay (41746 Rows).

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







