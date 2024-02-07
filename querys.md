UPDATE customer_orders SET exclusions =NULL, extras = Null WHERE (exclusions='') AND (extras='');
UPDATE customer_orders SET exclusions =NULL, extras = Null WHERE (exclusions='null') AND (extras='null');
UPDATE customer_orders SET exclusions =NULL WHERE (exclusions='null') or (exclusions='');
UPDATE customer_orders SET extras =NULL WHERE (extras='null') or (extras='');
UPDATE runner_orders SET cancellation = NULL WHERE (cancellation = '') OR (cancellation = 'null');
UPDATE runner_orders SET pickup_time = NULL, distance = NULL, duration = NULL WHERE (pickup_time = 'null') AND (distance = 'null') AND ( duration = 'null');

## A. Pizza Metrics

How many pizzas were ordered? : 

```
SELECT COUNT(order_id) FROM customer_orders;
```

How many unique customer orders were made? : 


```
SELECT COUNT(DISTINCT customer_id) FROM customer_orders;
```


How many successful orders were delivered by each runner? : 

```
SELECT runner_id, COUNT(order_id) as order_count FROM runner_orders WHERE cancellation IS NULL GROUP BY runner_id;
```

How many of each type of pizza was delivered?  : 

```
SELECT pizza_id, COUNT(pizza_id) as count_pizza FROM customer_orders GROUP BY pizza_id ORDER BY pizza_id;
```

How many Vegetarian and Meatlovers were ordered by each customer? : 

```
SELECT customer_id, pizza_id, COUNT(order_id) AS number_of_orders FROM customer_orders WHERE pizza_id IN (1, 2) GROUP BY customer_id, pizza_id;
```

What was the maximum number of pizzas delivered in a single order? : 

```
SELECT order_id, pizza_id, COUNT(order_id) AS number_of_orders FROM customer_orders WHERE pizza_id IN (1, 2) GROUP BY order_id, pizza_id ORDER BY number_of_orders DESC LIMIT 1;
```

For each customer, how many delivered pizzas had at least 1 change and how many had no changes? : 

```
SELECT customer_id, COUNT(CASE WHEN exclusions is NOT NULL OR extras is NOT NULL THEN 1 END) AS count_of_changes, COUNT(CASE WHEN exclusions is NULL or extras is NULL THEN 1 END) AS count_of_no_changes FROM customer_orders GROUP BY customer_id;
```

How many pizzas were delivered that had both exclusions and extras? : 

```
SELECT COUNT(CASE WHEN exclusions is NOT NULL AND extras is NOT NULL THEN 1 END) AS number_of_order_with_exclusion_and_extras FROM customer_orders;
```

What was the total volume of pizzas ordered for each hour of the day?

```
SELECT EXTRACT(HOUR FROM order_time) AS hour, COUNT(*) AS total_volume FROM customer_orders GROUP BY hour ORDER BY hour;
```

What was the volume of orders for each day of the week?

```
SELECT CASE  
WHEN EXTRACT(DOW FROM order_time) = 0 THEN 'Sunday'
WHEN EXTRACT(DOW FROM order_time) = 1 THEN 'monday'
WHEN EXTRACT(DOW FROM order_time) = 2 THEN 'Tuesday'
WHEN EXTRACT(DOW FROM order_time) = 3 THEN 'Wednesday'
WHEN EXTRACT(DOW FROM order_time) = 4 THEN 'Thursday'
WHEN EXTRACT(DOW FROM order_time) = 5 THEN 'Friday'
WHEN EXTRACT(DOW FROM order_time) = 6 THEN 'Saterday'
END AS day_of_week, COUNT(*) AS total_order_per_day_of_week
FROM customer_orders GROUP BY day_of_week;
```


## Runner and Customer Experience

How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01) :

```
SELECT EXTRACT(WEEK FROM registration_date) AS week, COUNT(*) AS number_of_registration_per_week FROM runners GROUP BY week;
```

What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order? :

```
SELECT runner_id, ROUND(AVG(EXTRACT(EPOCH FROM (pickup_time - order_time)) / 60), 2) AS avg_pickup_time_minutes FROM over_view WHERE pickup_time IS NOT NULL AND order_time IS NOT NULL GROUP BY runner_id;
```

Is there any relationship between the number of pizzas and how long the order takes to prepare? :

```
SELECT order_id, COUNT(order_id) AS pizza_number, ROUND(AVG(EXTRACT(EPOCH FROM (pickup_time - order_time)) / 60), 2) AS preparation_time_minutes FROM over_view WHERE order_time IS NOT NULL AND
pickup_time IS NOT NULL GROUP BY order_id ORDER BY order_id;
```

What was the average distance travelled for each customer? :

```
SELECT customer_id, ROUND(AVG(distance_in_meter ), 2) AS avg_distance_each_customer_meters FROM over_view WHERE distance_in_meter IS NOT NULL GROUP BY customer_id;
```

What was the difference between the longest and shortest delivery times for all orders? :

```
SELECT ROUND(EXTRACT(EPOCH FROM (MAX(duration) - MIN(duration)) / 60),2) AS Delivery_Time_Range_Min FROM over_view;
```

What was the average speed for each runner for each delivery and do you notice any trend for these values? :

```
SELECT runner_id, AVG(duration) as avg_delivery_time FROM runner_orders GROUP BY runner_id ORDER BY runner_id;
```

What is the successful delivery percentage for each runner? :

```
SELECT runner_id, COUNT(*) FILTER (WHERE cancellation IS NULL) / CAST(COUNT(*) AS FLOAT) * 100 AS success_percentage FROM runner_orders GROUP BY runner_id;
```


##  Ingredient Optimisation

What are the standard ingredients for each pizza?

```
SELECT pn.pizza_name, string_agg(sub.tp_name, ', ') AS standard_ingredients FROM pizza_names pn JOIN pizza_recipes pr ON pn.pizza_id = pr.pizza_id 
JOIN LATERAL(SELECT pt.topping_name AS tp_name FROM pizza_toppings pt JOIN UNNEST(pr.toppings) AS tp_num ON tp_num = pt.topping_id) sub ON TRUE
GROUP BY pn.pizza_name;

```

What was the most commonly added extra?

```
SELECT pt.topping_name, COUNT(*) AS frequency  FROM pizza_toppings pt 
JOIN LATERAL(SELECT extra FROM customer_orders co, UNNEST(co.extras) AS extra WHERE extras IS NOT NULL) sub ON  sub.extra = pt.topping_id
GROUP BY pt.topping_name ORDER BY frequency DESC;
```

What was the most common exclusion?

```
SELECT pt.topping_name, COUNT(*) AS frequency FROM pizza_toppings pt 
JOIN (SELECT exclusion FROM customer_orders co, UNNEST(co.exclusions) AS exclusion WHERE exclusions IS NOT NULL) sub ON sub.exclusion = pt.topping_id 
GROUP BY pt.topping_name ORDER BY frequency DESC;

```

Generate an order item for each record in the customers_orders table in the format of one of the following:

Meat Lovers
Meat Lovers - Exclude Beef
Meat Lovers - Extra Bacon
Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers

```
SELECT co.order_id, CASE WHEN co.pizza_id = 1 THEN '--Meat Lovers - ' ELSE '--Vegetarian - ' END || 
COALESCE( 'Exclude ' || sub.tp_name, '') ||
COALESCE( '- Extra ' || sub2.tp_name, '') AS pizza_exclude_extra
FROM customer_orders co 
LEFT JOIN LATERAL(SELECT STRING_AGG(pt.topping_name, ', ') AS tp_name FROM pizza_toppings pt 
JOIN UNNEST(co.exclusions) AS exclusion ON pt.topping_id = exclusion) sub ON TRUE
LEFT JOIN LATERAL(SELECT STRING_AGG(pt.topping_name, ', ') AS tp_name FROM pizza_toppings pt 
JOIN UNNEST(co.extras) AS extra ON pt.topping_id = extra) sub2 ON TRUE
GROUP BY  co.order_id, co.pizza_id, sub.tp_name, sub2.tp_name;

```

Generate an alphabetically ordered comma separated ingredient list for each pizza order from the customer_orders table and add a 2x in front of any relevant ingredients

For example: "Meat Lovers: 2xBacon, Beef, ... , Salami"

```
SELECT co.order_id, CASE WHEN co.pizza_id = 1 THEN 'Meat lovers: ' ELSE 'Vegetarian: ' END || 
CASE WHEN s.extra_count = 2 THEN ' 2x ' ELSE '' END ||
STRING_AGG(s.name_topping, ', ') AS pizza_and_extras
FROM customer_orders co 
JOIN LATERAL(SELECT pt.topping_name AS name_topping, COUNT(*) AS extra_count 
FROM UNNEST(co.extras) AS extra_topping JOIN pizza_toppings pt 
ON pt.topping_id = extra_topping GROUP BY pt.topping_name) s ON TRUE
GROUP BY co.order_id, co.pizza_id, s.extra_count;
```


What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?

```
SELECT pt.topping_name, COUNT(*) AS Amount 
FROM pizza_toppings pt
JOIN LATERAL(SELECT UNNEST(pr.toppings) 
AS topping_number FROM pizza_recipes pr
JOIN customer_orders co ON pr.pizza_id = co.pizza_id) AS test
ON pt.topping_id = test.topping_number
GROUP BY pt.topping_name
ORDER BY Amount DESC;

```


SELECT 


## D. Pricing and Ratings


1. If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?

```
SELECT CASE WHEN co.pizza_id = 1 THEN 'Meat Lovers' ELSE 'Vegetarian' END AS pizza_name,
COUNT(*) * sub.price AS total_revenue
FROM customer_orders co 
LEFT JOIN runner_orders ro ON co.order_id = ro.id 
JOIN LATERAL(SELECT CASE WHEN co.pizza_id = 1 THEN 12 ELSE 10 END AS price) sub ON TRUE
WHERE ro.cancellation IS NULL
GROUP BY pizza_name, sub.price;

```

2. What if there was an additional $1 charge for any pizza extras?
    * Add cheese is $1 extra


```
SELECT pn.pizza_name, SUM(sub.pizza_price + COALESCE(sub2.extra_count, 0)) AS pizza_price
FROM customer_orders co JOIN pizza_names pn ON pn.pizza_id = co.pizza_id 
JOIN runner_orders ro ON co.order_id = ro.id 
JOIN LATERAL(SELECT CASE WHEN co.pizza_id = 1 THEN 12 ELSE 10 END AS pizza_price) sub ON TRUE 
JOIN LATERAL(SELECT COUNT(*) AS extra_count FROM UNNEST(co.extras) AS extra) sub2 ON TRUE
WHERE ro.cancellation IS NULL
GROUP BY pn.pizza_name;

```
    
3. The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional table for this new dataset - generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.

```

```

4. Using your newly generated table - can you join all of the information together to form a table which has the following information for successful deliveries?
* `customer_id`
* `order_id`
* `runner_id`
* `rating`
* `order_time`
* `pickup_time`
* Time between order and pickup
* Delivery duration
* Average speed
* Total number of pizzas

```

```
    
5. If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometre traveled - how much money does Pizza Runner have left over after these deliveries?

```

```
  
**Bonus Questions**
   
If Danny wants to expand his range of pizzas - how would this impact the existing data design? Write an `INSERT` statement to demonstrate what would happen if a new `Supreme` pizza with all the toppings was added to the Pizza Runner menu?
