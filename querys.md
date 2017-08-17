---
The idea is collect all sql's queries from discourse.org, quote the sources!
For codes:
```sql
code
```
For sources:
[Source](url)

Remember:
Queries are limited to a 10 second runtime, plus SET TRANSACTION READ ONLY.
---

# List of Queries

## Fetch top 10 posts by likes received in the last month
>Fetch top 10 posts by likes received in the last month, excluding administrators. If the likes count equals, prioritize the posts that were created earlier. ([Source](https://meta.discourse.org/t/data-explorer-plugin/32566/9?u=sidv))

```sql
SELECT p.id as post_id, p.like_count as like_count
FROM posts p
LEFT JOIN topics t ON t.id = p.topic_id
LEFT JOIN users u ON u.id = p.user_id
WHERE p.created_at >= CURRENT_DATE - INTERVAL '1 month'
  AND NOT u.admin
  AND NOT u.blocked
ORDER BY p.like_count DESC, p.created_at ASC
LIMIT 10
```

## Users Last Seen Since (Since N Weeks Ago)
[Source](https://meta.discourse.org/t/what-cool-data-explorer-queries-have-you-come-up-with/43516?u=sidv)
```sql
with intervals as (
    select
        n as start_time, 
        CURRENT_TIMESTAMP as end_time
    from generate_series(CURRENT_TIMESTAMP - INTERVAL '140 days',
                         CURRENT_TIMESTAMP - INTERVAL '7 days',
                         INTERVAL '7 days') n
)
select 
  COUNT(1) as users_seen_since,
  CURRENT_TIMESTAMP - i.start_time as time_ago
FROM USERS u
right join intervals i
on u.last_seen_at >= i.start_time and u.last_seen_at < i.end_time
group by i.start_time, i.end_time
order by i.start_time desc
```

## Users Last Seen Since (Since N Days Ago)
[Source](https://meta.discourse.org/t/what-cool-data-explorer-queries-have-you-come-up-with/43516?u=sidv)
```sql
with intervals as (
    select
        n as start_time, 
        CURRENT_TIMESTAMP as end_time
    from generate_series(CURRENT_TIMESTAMP - INTERVAL '30 days',
                         CURRENT_TIMESTAMP - INTERVAL '1 day',
                         INTERVAL '1 day') n
)
select 
  COUNT(1) as users_seen_since,
  CURRENT_TIMESTAMP - i.start_time as time_ago
FROM USERS u
right join intervals i
on u.last_seen_at >= i.start_time and u.last_seen_at < i.end_time
group by i.start_time, i.end_time
order by i.start_time desc
```

## Banner Stats 
[Source](https://meta.discourse.org/t/what-cool-data-explorer-queries-have-you-come-up-with/43516/2?u=sidv)
```sql
WITH all_users AS ( SELECT
  COUNT(users.id) AS user_count
  FROM users
  WHERE users.active 
  AND users.suspended_at IS NULL
  AND users.locale IS NOT NULL
)
, read_banner_topic AS ( SELECT 
  COUNT(topic_views.user_id) AS read_count
  FROM topic_views  
  WHERE topic_views.topic_id IN (
    SELECT topics.id 
    FROM topics 
    WHERE topics.archetype IS NOT NULL
    AND topics.archetype LIKE 'banner'
  )
  AND topic_views.user_id IS NOT NULL
)
, dismissed_banner AS ( SELECT 
  COUNT(user_profiles.user_id) AS dismissed_count
  FROM user_profiles
  WHERE user_profiles.dismissed_banner_key IS NOT NULL 
  AND user_profiles.dismissed_banner_key IN (
    SELECT topics.id 
    FROM topics 
    WHERE topics.archetype IS NOT NULL
    AND topics.archetype LIKE 'banner'
  )
  AND user_profiles.user_id IS NOT NULL
)
SELECT all_users.user_count
  , read_banner_topic.read_count
  , dismissed_banner.dismissed_count 
FROM all_users, read_banner_topic, dismissed_banner
```
