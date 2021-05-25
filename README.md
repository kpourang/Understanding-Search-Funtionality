<p>The product team is determining priorities for the next development cycle and they are considering improving the site's search functionality. It currently works as follows:<p>

<ul>
    <li>There is a search box in the header the persists on every page of the website. It prompts users to search for people, groups, and conversations.</li>
    <img src = "https://github.com/kpourang/Understanding-Search-Funtionality/blob/main/yammer-search.png">
    <li>When a user begins to type in the search box, a dropdown list with the most relevant results appears. The results are separated by category (people, conversations, files, etc.). There is also an option to view all results.</li>
    <img src = "https://github.com/kpourang/Understanding-Search-Funtionality/blob/main/yammer-search-autocomplete.png">
    <li>When the user hits enter or selects “view all results” from the dropdown, she is taken to a results page, with results separated by tabs for different categories (people, conversations, etc.). Each tab is order by relevance and chronology (more recent posts surface higher).</li>
    <img src = "https://github.com/kpourang/Understanding-Search-Funtionality/blob/main/yammer-search-results.png">
    <li>The search results page also has an “advanced search” box that allows the user to search again within a specific Yammer group or date range.</li>
</ul>

<h1>The problem</h1>

<p>Before tackling search, the product team wants to make sure that the engineering team's time will be well-spent in doing so. After all, each new feature comes at the expense of some other potential feature(s). The product team is most interested in determining whether they should even work on search in the first place and, if so, how they should modify it.</p>

<h1>Getting oriented</h1>

<p>Before looking at the data, develop some hypotheses about how users might interact with search. What is the purpose of search? How would you know if it is fulfilling that purpose? How might you (quantitatively) understand the general quality of an individual user's search experience?</p>

<h1>The data</h1>

<p>There are two tables that are relevant to this problem. Most critically, there are certain events that you will want to look into in the events table below:</p>

<ul>
    <li>search_autocomplete: This is logged when a user clicks on a search option from autocomplete</li>
    <li>search_run: This is logged when a user runs a search and sees the search results page.</li>
    <li>search_click_X: This is logged when a user clicks on a search result. X, which ranges from 1 to 10, describes which search result was clicked.</li>

<p>The tables names and column definitions are listed below—click a table name to view information about that table.</p>

<h2>Table 1: Users</h2>
This table includes one row per user, with descriptive information about that user's account.
<img src = "https://github.com/kpourang/Understanding-Search-Funtionality/blob/main/Table1.JPG">

<h2>Table 2: Events</h2>
<p>This table includes one row per event, where an event is an action that a user has taken on Yammer. These events include login events, messaging events, search events, events logged as users progress through a signup funnel, events around received emails.</p>
<img src = "https://github.com/kpourang/Understanding-Search-Funtionality/blob/main/Table2.JPG">

<h1>Making a recommendation</h1>

<p>Once you have an understanding of the data, try to validate some of the hypotheses you formed earlier. In particular, you should seek to answer the following questions:</p>
<ul>
    <li>Are users' search experiences generally good or bad?</li>
    <li>Is search worth working on at all?</li>
    <li>If search is worth working on, what, specifically, should be improved?</li>

<p>Come up with a brief presentation describing the state of search at Yammer. Display your findings graphically. You should be prepared to recommend what, if anything, should be done to improve search. If you determine that you do not have sufficient information to test anything you deem relevant, discuss the caveats.</p>

<p>Finally, determine a way to understand whether your feature recommendations are actually improvements over the old search (assuming that anything you recommend will be completed).</p>

<h1> Solution</h1>
<h1>Developing hypotheses</h1>

<p>Framing problems simply and correctly can often save time later on. Thinking about the ultimate purpose of search right off the bat can make it easier to evaluate other parts of them problem. Search, at the most basic level, is about helping people find what they're looking for easily. A great search product achieves this quickly and with minimal work on behalf of the user.<p>

<p>To understand whether search is fulfilling that purpose, consider some possibilities:</p>
<ul>
    <li>Search use: The first thing to understand is whether anyone even uses search at all</li>
    <li>Search frequency: If users search a lot, it's likely that they're getting value out of the feature ‐ with a major exception. If users search repeatedly within a short timeframe, it's likely that they're refining their terms because they were unable to find what they wanted initially.</li>
    <li>Repeated terms: A better way to understand the above would be to actually compare similarity of search terms. That's much slower and more difficult to actually do than counting the number of searches a user performs in a short timeframe, so best to ignore this option.</li>
    <li>Clickthroughs: If a user clicks many links in the search results, it's likely that she isn't having a great experience. However, the inverse is not necessarily true—clicking only one result does not imply a success. If the user clicks through one result, then refines her search, that's certainly not a great experience, so search frequency is probably a better way to understand that piece of the puzzle. Clickthroughs are, however, very useful in determining whether search rankings are good. If users frequently click low results or scroll to additional pages, then the ranking algorithm should probably be adjusted.</li>
    <li>Autocomplete Clickthroughs: The autocomplete feature is certainly part of the equation, though its success should be measured separately to understand its role.</li>

<h1>The state of search</h1>

<p>The criteria above suggest that understanding search on a session by session basis is going to be important for this problem. So before seeking to understand whether search is good or bad, it would be wise to define a session for the purposes of this problem, both practically and in terms of the data. For the following solution, a session is defined as a string of events logged by a user without a 10-minute break between any two events. So if a user goes 10 minutes without logging an event, the session is ended and her next engagement will be considered a new session.</p>

<p>1. First, take a look at how often people search and whether that changes over time. Users take advantage of the autocomplete function more frequently than they actually run searches that take them to the search results page:</p>
<pre>


SELECT DATE_TRUNC('week',z.session_start) AS week,
       COUNT(*) AS sessions,
       COUNT(CASE WHEN z.autocompletes > 0 THEN z.session ELSE NULL END) AS with_autocompletes,
       COUNT(CASE WHEN z.runs > 0 THEN z.session ELSE NULL END) AS with_runs
  FROM (
SELECT x.session_start,
       x.session,
       x.user_id,
       COUNT(CASE WHEN x.event_name = 'search_autocomplete' THEN x.user_id ELSE NULL END) AS autocompletes,
       COUNT(CASE WHEN x.event_name = 'search_run' THEN x.user_id ELSE NULL END) AS runs,
       COUNT(CASE WHEN x.event_name LIKE 'search_click_%' THEN x.user_id ELSE NULL END) AS clicks
  FROM (
SELECT e.*,
       session.session,
       session.session_start
  FROM tutorial.yammer_events e
  LEFT JOIN (
       SELECT user_id,
              session,
              MIN(occurred_at) AS session_start,
              MAX(occurred_at) AS session_end
         FROM (
              SELECT bounds.*,
              		    CASE WHEN last_event >= INTERVAL '10 MINUTE' THEN id
              		         WHEN last_event IS NULL THEN id
              		         ELSE LAG(id,1) OVER (PARTITION BY user_id ORDER BY occurred_at) END AS session
                FROM (
                     SELECT user_id,
                            event_type,
                            event_name,
                            occurred_at,
                            occurred_at - LAG(occurred_at,1) OVER (PARTITION BY user_id ORDER BY occurred_at) AS last_event,
                            LEAD(occurred_at,1) OVER (PARTITION BY user_id ORDER BY occurred_at) - occurred_at AS next_event,
                            ROW_NUMBER() OVER () AS id
                       FROM tutorial.yammer_events e
                      WHERE e.event_type = 'engagement'
                      ORDER BY user_id,occurred_at
                     ) bounds
               WHERE last_event >= INTERVAL '10 MINUTE'
                  OR next_event >= INTERVAL '10 MINUTE'
               	 OR last_event IS NULL
              	 	 OR next_event IS NULL   
              ) final
        GROUP BY 1,2
       ) session
    ON e.user_id = session.user_id
   AND e.occurred_at >= session.session_start
   AND e.occurred_at <= session.session_end
 WHERE e.event_type = 'engagement'
       ) x
 GROUP BY 1,2,3
       ) z
 GROUP BY 1
 ORDER BY 1
</pre>
<img src = "https://github.com/kpourang/Understanding-Search-Funtionality/blob/main/Searches%20vs%20Autocomplete.png">

<p>To be more precise, autocomplete gets used in approximately 25% of sessions, while search is only used in 8% or so:</p>
<pre>


SELECT DATE_TRUNC('week',z.session_start) AS week, 
       COUNT(CASE WHEN z.autocompletes > 0 THEN z.session ELSE NULL END)/COUNT(*)::FLOAT AS with_autocompletes,
       COUNT(CASE WHEN z.runs > 0 THEN z.session ELSE NULL END)/COUNT(*)::FLOAT AS with_runs
  FROM (
SELECT x.session_start,
       x.session,
       x.user_id,
       COUNT(CASE WHEN x.event_name = 'search_autocomplete' THEN x.user_id ELSE NULL END) AS autocompletes,
       COUNT(CASE WHEN x.event_name = 'search_run' THEN x.user_id ELSE NULL END) AS runs,
       COUNT(CASE WHEN x.event_name LIKE 'search_click_%' THEN x.user_id ELSE NULL END) AS clicks
  FROM (
SELECT e.*,
       session.session,
       session.session_start
  FROM tutorial.yammer_events e
  LEFT JOIN (
       SELECT user_id,
              session,
              MIN(occurred_at) AS session_start,
              MAX(occurred_at) AS session_end
         FROM (
              SELECT bounds.*,
              		    CASE WHEN last_event >= INTERVAL '10 MINUTE' THEN id
              		         WHEN last_event IS NULL THEN id
              		         ELSE LAG(id,1) OVER (PARTITION BY user_id ORDER BY occurred_at) END AS session
                FROM (
                     SELECT user_id,
                            event_type,
                            event_name,
                            occurred_at,
                            occurred_at - LAG(occurred_at,1) OVER (PARTITION BY user_id ORDER BY occurred_at) AS last_event,
                            LEAD(occurred_at,1) OVER (PARTITION BY user_id ORDER BY occurred_at) - occurred_at AS next_event,
                            ROW_NUMBER() OVER () AS id
                       FROM tutorial.yammer_events e
                      WHERE e.event_type = 'engagement'
                      ORDER BY user_id,occurred_at
                     ) bounds
               WHERE last_event >= INTERVAL '10 MINUTE'
                  OR next_event >= INTERVAL '10 MINUTE'
               	 OR last_event IS NULL
              	 	 OR next_event IS NULL   
              ) final
        GROUP BY 1,2
       ) session
    ON e.user_id = session.user_id
   AND e.occurred_at >= session.session_start
   AND e.occurred_at <= session.session_end
 WHERE e.event_type = 'engagement'
       ) x
 GROUP BY 1,2,3
       ) z
 GROUP BY 1
 ORDER BY 1
</pre>
<img src = "https://github.com/kpourang/Understanding-Search-Funtionality/blob/main/Percent%20of%20Sessions%20with%20Search%20Runs%20and%20Autocompeltes.png">

<p>Autocomplete's 25% use indicates that there is a need for users to find information on their Yammer networks. In other words, it's a feature that people use and is worth some attention.</p>

<p>2. As you can see below, autocomplete is typically used once or twice per session:</p>
<pre>


SELECT autocompletes,
       COUNT(*) AS sessions
  FROM (
SELECT x.session_start,
       x.session,
       x.user_id,
       COUNT(CASE WHEN x.event_name = 'search_autocomplete' THEN x.user_id ELSE NULL END) AS autocompletes,
       COUNT(CASE WHEN x.event_name = 'search_run' THEN x.user_id ELSE NULL END) AS runs,
       COUNT(CASE WHEN x.event_name LIKE 'search_click_%' THEN x.user_id ELSE NULL END) AS clicks
  FROM (
SELECT e.*,
       session.session,
       session.session_start
  FROM tutorial.yammer_events e
  LEFT JOIN (
       SELECT user_id,
              session,
              MIN(occurred_at) AS session_start,
              MAX(occurred_at) AS session_end
         FROM (
              SELECT bounds.*,
              		    CASE WHEN last_event >= INTERVAL '10 MINUTE' THEN id
              		         WHEN last_event IS NULL THEN id
              		         ELSE LAG(id,1) OVER (PARTITION BY user_id ORDER BY occurred_at) END AS session
                FROM (
                     SELECT user_id,
                            event_type,
                            event_name,
                            occurred_at,
                            occurred_at - LAG(occurred_at,1) OVER (PARTITION BY user_id ORDER BY occurred_at) AS last_event,
                            LEAD(occurred_at,1) OVER (PARTITION BY user_id ORDER BY occurred_at) - occurred_at AS next_event,
                            ROW_NUMBER() OVER () AS id
                       FROM tutorial.yammer_events e
                      WHERE e.event_type = 'engagement'
                      ORDER BY user_id,occurred_at
                     ) bounds
               WHERE last_event >= INTERVAL '10 MINUTE'
                  OR next_event >= INTERVAL '10 MINUTE'
               	 OR last_event IS NULL
              	 	 OR next_event IS NULL   
              ) final
        GROUP BY 1,2
       ) session
    ON e.user_id = session.user_id
   AND e.occurred_at >= session.session_start
   AND e.occurred_at <= session.session_end
 WHERE e.event_type = 'engagement'
       ) x
 GROUP BY 1,2,3
       ) z
 WHERE autocompletes > 0
 GROUP BY 1
 ORDER BY 1
</pre>
<img src = "https://github.com/kpourang/Understanding-Search-Funtionality/blob/main/Number%20of%20Sessions%20with%20Autocompletes.png">

<p>When users do run full searches, they typically run multiple searches in a single session. Considering full search is a more rarely used feature, this suggests that either the search results are not very good or that there is a very small group of users who like search and use it all the time:</p>
<pre>


SELECT runs,
       COUNT(*) AS sessions
  FROM (
SELECT x.session_start,
       x.session,
       x.user_id,
       COUNT(CASE WHEN x.event_name = 'search_autocomplete' THEN x.user_id ELSE NULL END) AS autocompletes,
       COUNT(CASE WHEN x.event_name = 'search_run' THEN x.user_id ELSE NULL END) AS runs,
       COUNT(CASE WHEN x.event_name LIKE 'search_click_%' THEN x.user_id ELSE NULL END) AS clicks
  FROM (
SELECT e.*,
       session.session,
       session.session_start
  FROM tutorial.yammer_events e
  LEFT JOIN (
       SELECT user_id,
              session,
              MIN(occurred_at) AS session_start,
              MAX(occurred_at) AS session_end
         FROM (
              SELECT bounds.*,
              		    CASE WHEN last_event >= INTERVAL '10 MINUTE' THEN id
              		         WHEN last_event IS NULL THEN id
              		         ELSE LAG(id,1) OVER (PARTITION BY user_id ORDER BY occurred_at) END AS session
                FROM (
                     SELECT user_id,
                            event_type,
                            event_name,
                            occurred_at,
                            occurred_at - LAG(occurred_at,1) OVER (PARTITION BY user_id ORDER BY occurred_at) AS last_event,
                            LEAD(occurred_at,1) OVER (PARTITION BY user_id ORDER BY occurred_at) - occurred_at AS next_event,
                            ROW_NUMBER() OVER () AS id
                       FROM tutorial.yammer_events e
                      WHERE e.event_type = 'engagement'
                      ORDER BY user_id,occurred_at
                     ) bounds
               WHERE last_event >= INTERVAL '10 MINUTE'
                  OR next_event >= INTERVAL '10 MINUTE'
               	 OR last_event IS NULL
              	 	 OR next_event IS NULL   
              ) final
        GROUP BY 1,2
       ) session
    ON e.user_id = session.user_id
   AND e.occurred_at >= session.session_start
   AND e.occurred_at <= session.session_end
 WHERE e.event_type = 'engagement'
       ) x
 GROUP BY 1,2,3
       ) z
 WHERE runs > 0
 GROUP BY 1
 ORDER BY 1
</pre>
<img src = "https://github.com/kpourang/Understanding-Search-Funtionality/blob/main/Number%20of%20Sessions%20with%20Runs.png">

<p>3. Digging in a bit deeper, it's clear that search isn't performing particularly well. In sessions during which users do search, they almost never click any of the results:</p>
<pre>


SELECT clicks,
       COUNT(*) AS sessions
  FROM (
SELECT x.session_start,
       x.session,
       x.user_id,
       COUNT(CASE WHEN x.event_name = 'search_autocomplete' THEN x.user_id ELSE NULL END) AS autocompletes,
       COUNT(CASE WHEN x.event_name = 'search_run' THEN x.user_id ELSE NULL END) AS runs,
       COUNT(CASE WHEN x.event_name LIKE 'search_click_%' THEN x.user_id ELSE NULL END) AS clicks
  FROM (
SELECT e.*,
       session.session,
       session.session_start
  FROM tutorial.yammer_events e
  LEFT JOIN (
       SELECT user_id,
              session,
              MIN(occurred_at) AS session_start,
              MAX(occurred_at) AS session_end
         FROM (
              SELECT bounds.*,
              		    CASE WHEN last_event >= INTERVAL '10 MINUTE' THEN id
              		         WHEN last_event IS NULL THEN id
              		         ELSE LAG(id,1) OVER (PARTITION BY user_id ORDER BY occurred_at) END AS session
                FROM (
                     SELECT user_id,
                            event_type,
                            event_name,
                            occurred_at,
                            occurred_at - LAG(occurred_at,1) OVER (PARTITION BY user_id ORDER BY occurred_at) AS last_event,
                            LEAD(occurred_at,1) OVER (PARTITION BY user_id ORDER BY occurred_at) - occurred_at AS next_event,
                            ROW_NUMBER() OVER () AS id
                       FROM tutorial.yammer_events e
                      WHERE e.event_type = 'engagement'
                      ORDER BY user_id,occurred_at
                     ) bounds
               WHERE last_event >= INTERVAL '10 MINUTE'
                  OR next_event >= INTERVAL '10 MINUTE'
               	 OR last_event IS NULL
              	 	 OR next_event IS NULL   
              ) final
        GROUP BY 1,2
       ) session
    ON e.user_id = session.user_id
   AND e.occurred_at >= session.session_start
   AND e.occurred_at <= session.session_end
 WHERE e.event_type = 'engagement'
       ) x
 GROUP BY 1,2,3
       ) z
 WHERE runs > 0
 GROUP BY 1
 ORDER BY 1
</pre>
<img src = "https://github.com/kpourang/Understanding-Search-Funtionality/blob/main/Clicks%20per%20Session%20with%20at%20Least%20One%20Search%20Run.png">

<p>Furthermore, more searches in a given session do not lead to many more clicks, on average:</p>
<pre>


SELECT runs,
       AVG(clicks)::FLOAT AS average_clicks
  FROM (
SELECT x.session_start,
       x.session,
       x.user_id,
       COUNT(CASE WHEN x.event_name = 'search_autocomplete' THEN x.user_id ELSE NULL END) AS autocompletes,
       COUNT(CASE WHEN x.event_name = 'search_run' THEN x.user_id ELSE NULL END) AS runs,
       COUNT(CASE WHEN x.event_name LIKE 'search_click_%' THEN x.user_id ELSE NULL END) AS clicks
  FROM (
SELECT e.*,
       session.session,
       session.session_start
  FROM tutorial.yammer_events e
  LEFT JOIN (
       SELECT user_id,
              session,
              MIN(occurred_at) AS session_start,
              MAX(occurred_at) AS session_end
         FROM (
              SELECT bounds.*,
              		    CASE WHEN last_event >= INTERVAL '10 MINUTE' THEN id
              		         WHEN last_event IS NULL THEN id
              		         ELSE LAG(id,1) OVER (PARTITION BY user_id ORDER BY occurred_at) END AS session
                FROM (
                     SELECT user_id,
                            event_type,
                            event_name,
                            occurred_at,
                            occurred_at - LAG(occurred_at,1) OVER (PARTITION BY user_id ORDER BY occurred_at) AS last_event,
                            LEAD(occurred_at,1) OVER (PARTITION BY user_id ORDER BY occurred_at) - occurred_at AS next_event,
                            ROW_NUMBER() OVER () AS id
                       FROM tutorial.yammer_events e
                      WHERE e.event_type = 'engagement'
                      ORDER BY user_id,occurred_at
                     ) bounds
               WHERE last_event >= INTERVAL '10 MINUTE'
                  OR next_event >= INTERVAL '10 MINUTE'
               	 OR last_event IS NULL
              	 	 OR next_event IS NULL   
              ) final
        GROUP BY 1,2
       ) session
    ON e.user_id = session.user_id
   AND e.occurred_at >= session.session_start
   AND e.occurred_at <= session.session_end
 WHERE e.event_type = 'engagement'
       ) x
 GROUP BY 1,2,3
       ) z
 WHERE runs > 0
 GROUP BY 1
 ORDER BY 1
</pre>
<img src = "https://github.com/kpourang/Understanding-Search-Funtionality/blob/main/Average%20Clicks%20per%20Session%20by%20Searches%20per%20Session.png">

<p>4. When users do click on search results, their clicks are fairly evenly distributed across the result order, suggesting the ordering is not very good. If search were performing well, this would be heavily weighted toward the top two or three results:</p>
</pre>


SELECT TRIM('search_click_result_' FROM event_name)::INT AS search_result,
       COUNT(*) AS clicks
  FROM tutorial.yammer_events
 WHERE event_name LIKE 'search_click_%'
 GROUP BY 1
 ORDER BY 1
</pre>
<img src = "https://github.com/kpourang/Understanding-Search-Funtionality/blob/main/Clicks%20by%20Search%20Result.png">

<p>5. Finally, users who run full searches rarely do so again within the following month:</p>
<pre>


SELECT searches,
       COUNT(*) AS users
  FROM (
SELECT user_id,
       COUNT(*) AS searches
  FROM (
SELECT x.session_start,
       x.session,
       x.user_id,
       x.first_search,
       COUNT(CASE WHEN x.event_name = 'search_run' THEN x.user_id ELSE NULL END) AS runs
  FROM (
SELECT e.*,
       first.first_search,
       session.session,
       session.session_start
  FROM tutorial.yammer_events e
  JOIN (
       SELECT user_id,
              MIN(occurred_at) AS first_search
         FROM tutorial.yammer_events
        WHERE event_name = 'search_run'
        GROUP BY 1
       ) first
    ON first.user_id = e.user_id
   AND first.first_search <= '2014-08-01'
  LEFT JOIN (
       SELECT user_id,
              session,
              MIN(occurred_at) AS session_start,
              MAX(occurred_at) AS session_end
         FROM (
              SELECT bounds.*,
              		    CASE WHEN last_event >= INTERVAL '10 MINUTE' THEN id
              		         WHEN last_event IS NULL THEN id
              		         ELSE LAG(id,1) OVER (PARTITION BY user_id ORDER BY occurred_at) END AS session
                FROM (
                     SELECT user_id,
                            event_type,
                            event_name,
                            occurred_at,
                            occurred_at - LAG(occurred_at,1) OVER (PARTITION BY user_id ORDER BY occurred_at) AS last_event,
                            LEAD(occurred_at,1) OVER (PARTITION BY user_id ORDER BY occurred_at) - occurred_at AS next_event,
                            ROW_NUMBER() OVER () AS id
                       FROM tutorial.yammer_events e
                      WHERE e.event_type = 'engagement'
                      ORDER BY user_id,occurred_at
                     ) bounds
               WHERE last_event >= INTERVAL '10 MINUTE'
                  OR next_event >= INTERVAL '10 MINUTE'
               	 OR last_event IS NULL
              	 	 OR next_event IS NULL   
              ) final
        GROUP BY 1,2
       ) session
    ON session.user_id = e.user_id
   AND session.session_start <= e.occurred_at
   AND session.session_end >= e.occurred_at
   AND session.session_start <= first.first_search + INTERVAL '30 DAY'
 WHERE e.event_type = 'engagement'
       ) x
 GROUP BY 1,2,3,4
       ) z
 WHERE z.runs > 0
 GROUP BY 1
       ) z
 GROUP BY 1
 ORDER BY 1
LIMIT 100
</pre>
<img src = "https://github.com/kpourang/Understanding-Search-Funtionality/blob/main/Sessions%20with%20Session%20Runs%20in%20Month%20after%20User's%20First%20Search.png">

<p>Users who use the autocomplete feature, by comparison, continue to use it at a higher rate:</p>
<pre>


SELECT searches,
       COUNT(*) AS users
  FROM (
SELECT user_id,
       COUNT(*) AS searches
  FROM (
SELECT x.session_start,
       x.session,
       x.user_id,
       x.first_search,
       COUNT(CASE WHEN x.event_name = 'search_autocomplete' THEN x.user_id ELSE NULL END) AS autocompletes
  FROM (
SELECT e.*,
       first.first_search,
       session.session,
       session.session_start
  FROM tutorial.yammer_events e
  JOIN (
       SELECT user_id,
              MIN(occurred_at) AS first_search
         FROM tutorial.yammer_events
        WHERE event_name = 'search_autocomplete'
        GROUP BY 1
       ) first
    ON first.user_id = e.user_id
   AND first.first_search <= '2014-08-01'
  LEFT JOIN (
       SELECT user_id,
              session,
              MIN(occurred_at) AS session_start,
              MAX(occurred_at) AS session_end
         FROM (
              SELECT bounds.*,
              		    CASE WHEN last_event >= INTERVAL '10 MINUTE' THEN id
              		         WHEN last_event IS NULL THEN id
              		         ELSE LAG(id,1) OVER (PARTITION BY user_id ORDER BY occurred_at) END AS session
                FROM (
                     SELECT user_id,
                            event_type,
                            event_name,
                            occurred_at,
                            occurred_at - LAG(occurred_at,1) OVER (PARTITION BY user_id ORDER BY occurred_at) AS last_event,
                            LEAD(occurred_at,1) OVER (PARTITION BY user_id ORDER BY occurred_at) - occurred_at AS next_event,
                            ROW_NUMBER() OVER () AS id
                       FROM tutorial.yammer_events e
                      WHERE e.event_type = 'engagement'
                      ORDER BY user_id,occurred_at
                     ) bounds
               WHERE last_event >= INTERVAL '10 MINUTE'
                  OR next_event >= INTERVAL '10 MINUTE'
               	 OR last_event IS NULL
              	 	 OR next_event IS NULL   
              ) final
        GROUP BY 1,2
       ) session
    ON session.user_id = e.user_id
   AND session.session_start <= e.occurred_at
   AND session.session_end >= e.occurred_at
   AND session.session_start <= first.first_search + INTERVAL '30 DAY'
 WHERE e.event_type = 'engagement'
       ) x
 GROUP BY 1,2,3,4
       ) z
 WHERE z.autocompletes > 0
 GROUP BY 1
       ) z
 GROUP BY 1
 ORDER BY 1
LIMIT 100
</pre>
<img src = "https://github.com/kpourang/Understanding-Search-Funtionality/blob/main/new%20image.png">

<h1>Follow through</h1>

<p>This all suggests that autocomplete is performing reasonably well, while search runs are not. The most obvious place to focus is on the ordering of search results. It's important to consider that users likely run full searches when autocomplete does not provide the things they are looking for, so maybe changing the search ranking algorithm to provide results that are a bit different from the autocomplete results would help. Of course, there are many ways to approach the problem—the important thing is that the focus should be on improving full search results.</p>


