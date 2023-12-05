# Metro_car_funnel_analysis
Funnel analysis for UX for rideshare ap to identify where user churn arises and highlight areas of improvement in the UX. 

#### How many times was the app downloaded?
SELECT COUNT (distinct app_download_key)
FROM app_downloads;

23608

#### How many users signed up on the app?
SELECT COUNT (distinct user_id)
FROM signups;

17623

#### How many rides were requested through the app?
SELECT COUNT (distinct ride_id)
FROM ride_requests;

385477

#### How many rides were requested and completed through the app?
SELECT COUNT (request_ts) AS request, COUNT (dropoff_ts) AS dropoff
FROM ride_requests;

385477, 223652

#### How many rides were requested and how many unique users requested a ride?
SELECT COUNT (request_ts) AS requests, COUNT (distinct user_id) AS users
FROM ride_requests;

385477, 12406

#### What is the average time of a ride from pick up to drop off?
SELECT AVG(EXTRACT(EPOCH FROM ((dropoff_ts - pickup_ts)))) / 60  AS 	average_journey_time
FROM ride_requests; 

52 minutes

#### How many rides were accepted by a driver?
SELECT COUNT (accept_ts)
FROM ride_requests
WHERE accept_ts IS NOT NULL;

248379 

#### How many rides did we successfully collect payments and how much was collected?
SELECT COUNT (charge_status), SUM(purchase_amount_usd)
FROM transactions
WHERE charge_status = 'Approved';

212628, 4251667.61

#### How many ride requests happened on each platform?
SELECT platform, COUNT (r.ride_id) AS total
FROM app_downloads a 
LEFT JOIN signups s
ON a.app_download_key = s.session_id
  LEFT JOIN ride_requests r 
ON s.user_id = r.user_id
GROUP BY platform;

ios 234693, web 38467, android 112317

#### What is the drop-off from users signing up to users requesting a ride?
SELECT
  COUNT(distinct s.user_id) AS signups,
  COUNT(distinct r.user_id) AS ride_requests,
  COUNT(distinct s.user_id) - COUNT(distinct r.user_id) AS droppoff_count,
  1 - (
    COUNT(distinct r.user_id) / COUNT(distinct s.user_id)::numeric
  ) AS dropoff_rate
FROM
  signups s
  LEFT JOIN ride_requests r USING (user_id);

29.6%
