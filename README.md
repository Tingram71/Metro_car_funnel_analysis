# Metro_car_funnel_analysis
## Funnel analysis for UX for rideshare ap to identify where user churn arises and highlight areas of improvement in the UX. 

#### How many times was the app downloaded?
```sql
SELECT count(*) AS total_downloads
FROM app_downloads;

| total_downloads |
| --------------- |
| 23608           |
```



#### How many users signed up on the app?
```sql
SELECT COUNT(user_id) AS total_signups
FROM signups;

| total_signups |
| ------------- |
| 17623         |
````
#### How many rides were requested through the app?
```SQL
SELECT COUNT(request_ts) AS total_ride_requests
FROM ride_requests;

| total_ride_requests |
| ------------------- |
| 385477              |
````
#### How many rides were requested and completed through the app?
````sql
SELECT
  COUNT(request_ts) AS total_ride_requests,
  COUNT(dropoff_ts) AS total_completed_rides
FROM
  ride_requests;

| total_ride_requests | total_completed_rides |
| ------------------- | --------------------- |
| 385477              | 223652                |
`````
#### How many rides were requested and how many unique users requested a ride?
```sql
SELECT
  COUNT(request_ts) AS total_rides_requested,
  COUNT(DISTINCT user_id) AS total_unique_users
FROM
  ride_requests;

| total_rides_requested | total_unique_users |
| --------------------- | ------------------ |
| 385477                | 12406              |
````
#### What is the average time of a ride from pick up to drop off?
```sql
SELECT
  AVG(dropoff_ts - pickup_ts) AS avg_ride_duration
FROM
  ride_requests;

| avg_ride_duration |
| ----------------- |
| 00:52:36.738773   |
````
#### How many rides were accepted by a driver?
````sql
SELECT
  COUNT(accept_ts) AS total_rides_driver_accepted
FROM
  ride_requests;

| total_rides_driver_accepted |
| --------------------------- |
| 248379                      |
````
#### How many rides did we successfully collect payments and how much was collected?
````sql
SELECT
  COUNT(transaction_id) AS total_transactions,
  SUM(purchase_amount_usd) AS total_revenue
FROM
  transactions
WHERE
  charge_status = 'Approved';

| transaction_count | total_revenue     |
| ----------------- | ----------------- |
| 212628            | 4251667.609999995 |
````
#### How many ride requests happened on each platform?
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

| platform | total_ride_requests |
| -------- | ------------------- |
| ios      | 234693              |
| android  | 112317              |
| web      | 38467               |
````
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

| total_signups | total_ride_requests | total_droppoffs | dropoff_rate           |
| ------------- | ------------------- | --------------- | ---------------------- |
| 17623         | 12406               | 5217            | 0.29603359246439312262 |
````
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
| total_signups | total_rides_completed | percentage_completed |
| ------------- | --------------------- | -------------------- |
| 17623         | 6233                  | 35.3685524598536     |
````

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
| funnel_step | funnel_name    | value | previous_value     |
| ----------- | -------------- | ----- | ------------------ |
| 0           | downloads      | 23608 |                    |
| 1           | signups        | 17623 | 0.7464842426296171 |
| 2           | ride_requested | 12406 | 0.7039664075356069 |
````
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

| funnel_step | funnel_name    | value | top_value          |
| ----------- | -------------- | ----- | ------------------ |
| 0           | downloads      | 23608 | 1                  |
| 1           | signups        | 17623 | 0.7464842426296171 |
| 2           | ride_requested | 12406 | 0.5254998305659099 |
````
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
| funnel_step | funnel_name    | value | previous_value     |
| ----------- | -------------- | ----- | ------------------ |
| 0           | downloads      | 23608 |                    |
| 1           | signups        | 17623 | 0.7464842426296171 |
| 2           | ride_requested | 12406 | 0.7039664075356069 |
| 3           | ride_completed | 6233  | 0.5024181847493149 |

````
 #### Using the percent of top approach, what are the  user-level conversion rates for the following 3 stages of the funnel? 1. signup, 2. ride requested, 3. ride completed (hint: signup is the top of this funnel)
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

| funnel_step | funnel_name    | value | top_value     |
| ----------- | -------------- | ----- | ------------------ |
| 1           | signups        | 17623 | 1                  |
| 2           | ride_requested | 12406 | 0.7039664075356069 |
| 3           | ride_completed | 6233  | 0.353685524598536  |
````
#### Funnel Percent of Top:
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
,
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
  value::float / FIRST_VALUE(value) OVER (
    ORDER BY
      funnel_step
  ) AS top_value
FROM
  funnel_stages
ORDER BY
  funnel_step;
´´´´
