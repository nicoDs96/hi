---
title: "Dates intersection on SCD columns like a pro"
layout: post
date: 2022-02-21 23:57
tag: 
- query
- SCD
- dates
- segments
- intersection
image: /assets/images/time_seg_intersection/3.png
headerImage: false
star: true
usemathjax: true
projects: false
hidden: false # don't count this post in blog pagination
description: "Use logic to write queries conditions. Negate the problem to get a different perspective"
category: blog
author: nicods
---


Working as Data Engineer makes you work with dates and time data a lot. Especially in the recent period where companies want to be Data-Driven, decisions are data-driven, your coffee machine is data-driven, and AI and ML require tons of data to work.

In a data warehouse or data lake context, modeling data using a type of [slow changing dimenion](https://en.wikipedia.org/wiki/Slowly_changing_dimension) (SCD) is a quite popular choice. 
For example, suppose you have clients registry data, and each time a client changes his shipping address then you set the end_date of the previous shipping address and the start_date of the new record with the timestamp the modification inside the address_tbl:  

|client_id|shipping_address|start_timestamp|end_timestamp|
|---------|----------------|---------------|-------------|
|client_1|383 Trusel DriveNorth Andover, MA 01845|2015-01-14 00:00:00|2017-05-11 10:00:00|
|client_1|91 Gates St. Valrico, FL 33594|2017-05-11 10:00:00|2999-01-01 23:59:59 |

Then you know that a client_1's current shipping_address is the one satisfying the condition 
```sql
 client_id=client_1 and
    address_tbl.start_date <= current_date < address_tbl.end_date 
```

Then you have another table called fav_payment_method telling your client's favourtie payment method:  

|client_id|start_timestamp|end_timestamp|payment_method|
|---------|---------------|-------------|--------------|
|client_1|2015-01-01 00:00:00|2016-01-01 00:00:00|at_delivery_time|
|client_1|2016-01-01 00:00:00|2018-01-01 10:00:00|paypal|
|client_1|2018-01-01 10:00:00|2019-01-01 23:00:00|credit card|
|client_1|2019-01-01 23:00:00|2999-01-01 23:59:59|paypal|

And, don't ask the BU why, but you have to associate to each client's shipping address his favorite payment method. How often do I see people doing that? 
Sometimes:
```sql
SELECT * 
FROM address_tbl at INNER JOIN fav_payment_method fpm
    ON at.client_id = fpm.client_id
WHERE   (at.start_timestamp >= fpm.start_timestamp AND 
        at.start_timestamp < fpm.end_timestamp) OR -- (1)
        (at.end_timestamp >= fpm.start_timestamp AND
        at.end_timestamp < fpm.end_timestamp)  -- (2)
```
That graphically is 
<img class="image" src="{{ site.url }}/assets/images/time_seg_intersection/1.png" alt="Cover Image"/>

And you can clearly see that there is a case that is not covered!

Other times:
```sql
SELECT * 
FROM address_tbl at INNER JOIN fav_payment_method fpm
    ON at.client_id = fpm.client_id
WHERE   (at.start_timestamp >= fpm.start_timestamp AND
        at.start_timestamp < fpm.end_timestamp) OR -- (1)
        (at.end_timestamp >= fpm.start_timestamp AND
        at.end_timestamp < fpm.end_timestamp) OR -- (2 )
        (fpm.start_timestamp >= at.start_timestamp AND
        fpm.start_timestamp < at.end_timestamp) OR -- (3)
        (fpm.end_timestamp >= at.start_timestamp AND
        fpm.end_timestamp < at.end_timestamp)  -- (4)
```
That is like the previous image plus other two cases (still with some uncovered options!)
<img class="image" src="{{ site.url }}/assets/images/time_seg_intersection/2.png" alt="Cover Image"/>

But indeed we can be less naive than this. It turns out that the problem is a 1D segment intersection problem. Segments are your ranges of dates and you only need to take the periods that intersect. But how can we write that? And can we do it simply?  
Well, we can say that two segments intersect if and only if they not(not intersect). Ah-ah, simple right? 

<img class="image" src="{{ site.url }}/assets/images/time_seg_intersection/3.png" alt="Cover Image"/>


from the picture we see that there isn't an intersection when: 
```sql
at.start_timestamp > fpm.end_timestamp or at.end_timestamp < fpm.start_timestamp
```
This also means that negating this condition we obtain not(not intersect)
```sql
at.start_timestamp <= fpm.end_timestamp and  at.end_timestamp >= fpm.start_timestamp
```

and we can finally write a simple, clean and concise query that does exactly what we wanted:
```sql
SELECT * 
FROM address_tbl at INNER JOIN fav_payment_method fpm
    ON at.client_id = fpm.client_id
WHERE   at.start_timestamp <= fpm.end_timestamp and  
        at.end_timestamp >= fpm.start_timestamp
```

|fpm.client_id|fpm.start_timestamp|fpm.end_timestamp|fpm.payment_method|at.client_id|at.shipping_address|at.start_timestamp|at.end_timestamp|
|-------------|-------------------|-----------------|------------------|-------------------|------------------|----------------|----------------|
|client_1|2015-01-01 00:00:00|2016-01-01 00:00:00|at_delivery_time|client_1|383 Trusel DriveNorth Andover, MA 01845|2015-01-14 00:00:00|2017-05-11 10:00:00|
|client_1|2016-01-01 00:00:00|2018-01-01 10:00:00|paypal|client_1|383 Trusel DriveNorth Andover, MA 01845|2015-01-14 00:00:00|2017-05-11 10:00:00|
|client_1|2016-01-01 00:00:00|2018-01-01 10:00:00|paypal|client_1|91 Gates St. Valrico, FL 33594|2017-05-11 10:00:00|2999-01-01 23:59:59 |
|client_1|2018-01-01 10:00:00|2019-01-01 23:00:00|credit card|client_1|91 Gates St. Valrico, FL 33594|2017-05-11 10:00:00|2999-01-01 23:59:59 |
|client_1|2019-01-01 23:00:00|2999-01-01 23:59:59|paypal|client_1|91 Gates St. Valrico, FL 33594|2017-05-11 10:00:00|2999-01-01 23:59:59 |

## "Hey there are two start_timestamp and end_timestamp and they overlap, what should I do?"
So far so good, but then the final user of the tables says "Hey there are two start_timestamp and end_timestamp and they overlap, what should I do? Why are you data engineer always making my life difficult? I hate you guys!". It hurts our feelings each time we heard a complaint, so we go one step further and give them a unique, continuous, and coherent start_timestamp and end_timestamp. But How?  
<img class="image" src="{{ site.url }}/assets/images/time_seg_intersection/4.png" alt="Cover Image"/>
Looking at the picture We can compute the intersection of the date segments to get the new periods for each record. The intersection of 1D segments is $$ intersection\_start\_timestamp = max(ad.start\_timestamp, fpm.start\_timestamp)$$ and $$ intersection\_end\_timestamp = min(ad.end\_timestamp, fpm.end\_timestamp)$$
<img class="image" src="{{ site.url }}/assets/images/time_seg_intersection/5.gif" alt="Cover Image"/>
and the final result is:
```sql
SELECT fpm.client_id, 
    greatest(at.start_timestamp, fpm.start_timestamp) as start_timestamp,
    least(at.end_timestamp, fpm.end_timestamp) as end_timestamp,
    fpm.payment_method,
    at.shipping_address
FROM address_tbl at INNER JOIN fav_payment_method fpm
    ON at.client_id = fpm.client_id
WHERE   at.start_timestamp <= fpm.end_timestamp and  
        at.end_timestamp >= fpm.start_timestamp
```

|client_id|start_timestamp|end_timestamp|payment_method|shipping_address|
|-------------|-------------------|-----------------|------------------|-------------------|
|client_1|2015-01-01 00:00:00|2016-01-01 00:00:00|at_delivery_time|383 Trusel DriveNorth Andover, MA 01845|
|client_1|2016-01-01 00:00:00|2017-05-11 10:00:00|paypal|383 Trusel DriveNorth Andover, MA 01845|
|client_1|2017-05-11 10:00:00|2018-01-01 10:00:00|paypal|91 Gates St. Valrico, FL 33594|
|client_1|2018-01-01 10:00:00|2019-01-01 23:00:00|credit card|91 Gates St. Valrico, FL 33594|
|client_1|2019-01-01 23:00:00|2999-01-01 23:59:59|paypal|91 Gates St. Valrico, FL 33594|

Have a good query!