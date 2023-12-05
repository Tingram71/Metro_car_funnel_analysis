# Metro_car_funnel_analysis
Funnel analysis for UX for rideshare ap to identify where user churn arises and highlight areas of improvement in the UX. 

#### How many times was the app downloaded?
SELECT count(*) AS total_downloads
FROM app_downloads;

| total_downloads |
| --------------- |
| 23608           |



#### How many users signed up on the app?
SELECT COUNT(user_id) AS total_signups
FROM signups;

| total_signups |
| ------------- |
| 17623         |

#### How many rides were requested through the app?
SELECT COUNT(request_ts) AS total_ride_requests
FROM ride_requests;

| total_ride_requests |
| ------------------- |
| 385477              |

#### How many rides were requested and completed through the app?
SELECT
  COUNT(request_ts) AS total_ride_requests,
  COUNT(dropoff_ts) AS total_completed_rides
FROM
  ride_requests;

| total_ride_requests | total_completed_rides |
| ------------------- | --------------------- |
| 385477              | 223652                |

#### How many rides were requested and how many unique users requested a ride?
SELECT
  COUNT(request_ts) AS total_rides_requested,
  COUNT(DISTINCT user_id) AS total_unique_users
FROM
  ride_requests;

| total_rides_requested | total_unique_users |
| --------------------- | ------------------ |
| 385477                | 12406              |

#### What is the average time of a ride from pick up to drop off?
SELECT
  AVG(dropoff_ts - pickup_ts) AS avg_ride_duration
FROM
  ride_requests;

| avg_ride_duration |
| ----------------- |
| 00:52:36.738773   |

#### How many rides were accepted by a driver?
SELECT
  COUNT(accept_ts) AS total_rides_driver_accepted
FROM
  ride_requests;

| total_rides_driver_accepted |
| --------------------------- |
| 248379                      |

#### How many rides did we successfully collect payments and how much was collected?
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

#### How many ride requests happened on each platform?
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

#### What is the drop-off from users signing up to users requesting a ride?
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

29.6%
