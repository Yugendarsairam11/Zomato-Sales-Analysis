# Zomato-Sales-Analysis

use zomato;

# total amount each customer spent on the zomato

-- select 
-- s.userid,
-- sum(p.price) as total_spend
-- from 
-- users as u 
-- join 
-- sales as s 
-- on u.userid = s.userid
-- join 
-- product as p
-- on s.product_id = p.product_id
-- group by 1
-- order by 1

# How many days each customer stayed with zomato

-- select 
-- u.userid,
-- u.signup_date as sign_up_date,
-- max(s.created_date) as recent_transaction_date,
-- datediff(max(s.created_date), u.signup_date) + 1 as total_days_with_zomato_inclusive
-- from 
-- users as u 
-- join 
-- sales as s 
-- on u.userid = s.userid
-- join 
-- product as p
-- on s.product_id = p.product_id
-- group by 1,u.signup_date
-- order by s.userid

# How many days has each customer visited zomato

-- with fetch_data as (
-- select 
-- u.userid,
-- s.created_date,
-- lead(s.created_date) over(partition by u.userid order by s.created_date) as next_visited_date,
-- datediff(lead(s.created_date,1) over(partition by u.userid order by s.created_date),s.created_date) as difference_in_days
-- from 
-- users as u 
-- join 
-- sales as s 
-- on u.userid = s.userid
-- join 
-- product as p
-- on s.product_id = p.product_id
-- order by u.userid, s.created_date
-- )

-- SELECT userid,
--        AVG(difference_in_days) AS median
-- FROM (
--     SELECT userid,
--            difference_in_days,
--            (SELECT PERCENTILE_CALCULATION_SUBQUERY(0.1, difference_in_days) FROM fetch_data sub WHERE sub.userid = ud.userid) AS lower_bound,
--            (SELECT PERCENTILE_CALCULATION_SUBQUERY(0.9, difference_in_days) FROM fetch_data sub WHERE sub.userid = ud.userid) AS upper_bound
--     FROM fetch_data ud
-- ) AS filtered_data
-- WHERE difference_in_days >= lower_bound AND difference_in_days <= upper_bound
-- GROUP BY userid;

# first product brought by each customer

-- with fetch_data as (
-- select 
-- u.userid,
-- s.created_date,
-- s.product_id,
-- dense_rank() over(partition by u.userid order by s.created_date) as ranking_order
-- from 
-- users as u 
-- join 
-- sales as s 
-- on u.userid = s.userid
-- join 
-- product as p
-- order by u.userid, s.created_date
-- )

-- select 
-- *
-- from 
-- fetch_data 
-- where ranking_order = 1

# what is the most purchased item on the menu and how many times was it purchased by all customers

-- with fetch_data as (
-- # This query generates product id and number of times users ordered it
-- # from them we are taking the most frequently ordered product
-- select 
-- p.product_id,
-- count(u.userid) as purchases
-- from users as u 
-- join 
-- sales as s 
-- on u.userid = s.userid
-- join 
-- product as p
-- on p.product_id = s.product_id
-- group by p.product_id
-- order by 2 desc 
-- limit 1
-- )

# After finding out the most frequently ordered product, now taking the count of how many times users has ordered it

-- select 
-- s.userid,
-- count(s.product_id) as no_of_times_purchases
-- from 
-- sales as s 
-- where s.product_id = (select product_id from fetch_data)
-- group by 1

# Which item is the most popular for each customer
# Precisely what is the favourite product for each customer

-- with fetch_data as (
-- select 
-- u.userid,
-- p.product_id,
-- p.product_name,
-- count(p.product_id) as each_product_purchase,
-- dense_rank() over(partition by u.userid order by count(p.product_id) desc ) as ranking_order
-- from 
-- users as u 
-- join 
-- sales as s 
-- on u.userid = s.userid
-- join 
-- product as p 
-- on s.product_id = p.product_id
-- group by u.userid,p.product_id, p.product_name
-- order by 1
-- )

-- # For each user these are the most ordered items (can consider them as favourite)
-- select 
-- userid,
-- product_id,
-- product_name
-- from 
-- fetch_data 
-- where ranking_order = 1

# Which item was purchased first by the customer after they became a Gold Member 
# try finding out the most expensive he bought after becoming gold member

-- with fetch_data as (
-- select 
-- g.userid,
-- g.gold_signup_date,
-- s.created_date,
-- s.product_id,
-- dense_rank() over(partition by g.userid order by s.created_date) as ranking_order 
-- from 
-- goldusers_signup as g 
-- join 
-- sales as s 
-- on g.userid = s.userid
-- where s.created_date > g.gold_signup_date
-- )

-- select 
-- userid,
-- gold_signup_date,
-- created_date,
-- product_id
-- from 
-- fetch_data 
-- where ranking_order = 1

# Which item as purchased just before joining gold membership

-- with fetch_data as (
-- select 
-- g.userid,
-- g.gold_signup_date,
-- s.created_date,
-- s.product_id,
-- dense_rank() over(partition by g.userid order by s.created_date desc) as ranking_order
-- from 
-- goldusers_signup as g
-- join 
-- sales as s
-- on g.userid = s.userid
-- where s.created_date < g.gold_signup_date
-- ) 

-- select 
-- userid,
-- gold_signup_date,
-- created_date,
-- product_id
-- from 
-- fetch_data 
-- where ranking_order = 1

# what are the total number of orders made and amount spend by each customer before becoming gold member

-- select 
-- g.userid,
-- count(s.created_date) as total_orders_made,
-- sum(p.price) as total_amount_spent
-- from 
-- goldusers_signup as g 
-- join 
-- sales as s
-- on g.userid = s.userid
-- join 
-- product as p
-- on p.product_id = s.product_id
-- where s.created_date < g.gold_signup_date
-- group by g.userid
-- order by 1

# points generation 

with fetch_data as (
select 
u.userid,
p.product_id,
sum(p.price) as total_amount_spent,
case when p.product_id = 1 then 5 when p.product_id = 2 then 2 when p.product_id = 3 then 5 else 0 end as points
from 
users as u 
join 
sales as s 
on s.userid = u.userid
join 
product as p
on p.product_id = s.product_id
group by u.userid, p.product_id
order by u.userid
)

-- select 
-- userid,
-- round(sum(total_amount_spent / points) * 2.5,1) as total_cashback_received
-- from 
-- fetch_data
-- group by 1

select 
product_id,
sum(total_amount_spent / points)  as total_cashback_received
from 
fetch_data
group by 1

# Checking out who ever the gold users, what are their overall purchases in first one year after joining gold 
# and for that one year he earns 5 points for every 10rs he spent on zomato
# so now what will be his overall amount and how much points each customer has earned on it

-- select 
-- g.userid,
-- sum(p.price) as total_price,
-- sum(price) / 2 as points_earned
-- from 
-- goldusers_signup as g 
-- join 
-- sales as s 
-- on g.userid = s.userid
-- join 
-- product as p
-- on s.product_id = p.product_id
-- where s.created_date between g.gold_signup_date and date_add(g.gold_signup_date, interval 1 year)

-- # Two ways of taking 1 year interval period
-- -- g.gold_signup_date + interval 1 year

-- group by 1
-- order by 3 desc

# ranking the customer only when they are gold members else putting them as na 

select 
*,
case when g.gold_signup_date is null then 'na' else dense_rank() over(partition by g.userid order by s.created_date desc) end  as ranking_order
from 
goldusers_signup as g 
right join 
sales as s 
on g.userid = s.userid and s.created_date >= g.gold_signup_date
