---
layout: post
shortenedlink: Analysing page view data in Neo4J
title: Analysing page view data in Neo4J
tags: [snowplow, neo4j, graph database, path analysis, cypher]
author: Nick
category: Analytics
---

In the [first post](/blog/2014/07/28/explorations-in-analyzing-web-event-data-in-graph-databases/) in this series, we established that graph databases may allow us to analyse event-based data in new ways. In the [second post](/blog/2014/07/30/loading-snowplow-web-event-data-into-graph-databases-for-pathing-analysis/), we walked through how to get Snowplow page view data into a Neo4J database in a structure that will allow for some new analytics. In this post, we're going to start running some queries to see what new insights we can find in the data.

Let's start with some easy analysis while we get used to using Cypher. Some of these simpler ideas could be easily queried directly in SQL; we're just interested in checking out how Cypher works for now.
















### How many visits were there to our home page?

We start by finding all of views that lead to the homepage. Using double dashes is shorthand for an edge, without specifying the relationship type (because in this case, we know that the only edges that go between views and pages are *object* edges).

<pre>
MATCH (view:View)--(home:Page { id: 'snowplowanalytics.com/' })
RETURN home, count(view);
</pre>

This returns a table just simply tells us the number of visitors to the home page:

<pre>
+--------------------------------------------------------+
| home                                     | count(view) |
+--------------------------------------------------------+
| Node[64293]{id:"snowplowanalytics.com/"} | 31491       |
+--------------------------------------------------------+
</pre>

Now we can look for 'bounces' - visitors who only went to the homepage and then left the site. For this, we start by matching the same patterns, but then limit them with a <tt>WHERE</tt> clause. I haven't specified a direction for the *prev* edge, because I want to find journeys that have no pages either before or after the home page. 

<pre>
MATCH (view:View)--(home:Page { id: 'snowplowanalytics.com/' })
WHERE NOT (view)-[:PREV]-()
RETURN home, count(view);
</pre>

<pre>
+--------------------------------------------------------+
| home                                     | count(view) |
+--------------------------------------------------------+
| Node[64293]{id:"snowplowanalytics.com/"} | 8154        |
+--------------------------------------------------------+
</pre>

So, of the 31,491 home page views, 8,154 were bounces. Now let's consider a more interesting question, albeit one that could be reasonably answered in SQL.

## What page were users on before arriving at the 'About' page?

(We're interested in our 'About' page because this has our contact details). Now we're specifying the destination, and we want to find the previous page the user was on. That means it's just a case of following an *prev* edge from the events connected to the 'About' page node.

Again, we start by specifying a pattern that ends in the 'About' page, and then aggregating the results:

<pre>
MATCH (contact:Page {id:"snowplowanalytics.com/about/index.html"})<-[:OBJECT]-(c_view)<-[:PREV]-(prev1)-[:OBJECT]->(p:Page) 
RETURN p.id, count(p)
ORDER BY count(p) DESC
LIMIT 10;
</pre>

This time, I've asked Neo4J to order the results in descending order, and limit them to the top 10. It took 183ms to return:

<pre>
+----------------------------------------------------------+
| p.id                                          | count(p) |
+----------------------------------------------------------+
| "snowplowanalytics.com/about/team.html"       | 1368     |
| "snowplowanalytics.com/about/index.html"      | 839      |
| "snowplowanalytics.com/product/index.html"    | 657      |
| "snowplowanalytics.com/"                      | 603      |
| "snowplowanalytics.com/technology/index.html" | 258      |
| "snowplowanalytics.com/blog.html"             | 240      |
| "snowplowanalytics.com/blog/index.html"       | 239      |
| "snowplowanalytics.com/analytics/index.html"  | 223      |
| "snowplowanalytics.com/services/index.html"   | 218      |
| "snowplowanalytics.com/about/users.html"      | 106      |
+----------------------------------------------------------+
</pre>

We can use the browser console to help visualize these patterns. Here's one example: the page on the bottom right is the 'About' page that we're interested in, and we can get the identity of the preceeding page by clicking on it (in the bottom left).

<p style="text-align:center"><img src="/assets/img/blog/2014/07/Neo4j-about-to-prev.png"></p>

We could easily amend it to find the page users were on two steps before they got to the contact page by adding an extra *prev* edge into the MATCH clause:

<pre>
MATCH (about:Page {id:"snowplowanalytics.com/about/index.html"})<-[:OBJECT]-(c_view)<-[:PREV]-(prev1)<-[:PREV]-(prev2)-[:OBJECT]->(p:Page) 
RETURN p.id, count(p)
ORDER BY count(p) DESC
LIMIT 10;
</pre>

As a shortcut, we can instruct Neo4J to follow two *prev* edges by writing <tt>[:PREV*2]</tt>:

<pre>
MATCH (about:Page {id:"snowplowanalytics.com/about/index.html"})<-[:OBJECT]-(c_view)<-[:PREV*2]-(prev2)-[:OBJECT]->(p:Page) 
RETURN p.id, count(p)
ORDER BY count(p) DESC
LIMIT 10;
</pre>

In either case, we get a result in around 300ms:

<pre>
+-------------------------------------------------------------+
| p.id                                             | count(p) |
+-------------------------------------------------------------+
| "snowplowanalytics.com/about/index.html"         | 1017     |
| "snowplowanalytics.com/product/index.html"       | 387      |
| "snowplowanalytics.com/services/index.html"      | 354      |
| "snowplowanalytics.com/"                         | 287      |
| "snowplowanalytics.com/technology/index.html"    | 269      |
| "snowplowanalytics.com/analytics/index.html"     | 256      |
| "snowplowanalytics.com/about/users.html"         | 218      |
| "snowplowanalytics.com/about/team.html"          | 175      |
| "snowplowanalytics.com/about/community.html"     | 150      |
| "snowplowanalytics.com/product/get-started.html" | 113      |
+-------------------------------------------------------------+
</pre>

You'll notice that lots of the journeys started on same page they finished on. That's because a lot of parts of journeys seem to consist of refreshes. Later, we'll look at how we can 'clean up' our graph to account for these, but for now, let's just exclude them from our search. We can do this by getting a list of event nodes in a path, and insisting that none of them point to the 'About' page.

<pre>
MATCH path = (about:Page {id:"snowplowanalytics.com/about/index.html"})<-[:OBJECT]-(c_view)<-[:PREV*2]-(prev2)-[:OBJECT]->(p:Page) 
WHERE none(
	v IN NODES(path)[2..LENGTH(path)-1]
	WHERE v.page = about.id
)
RETURN p.id, count(p)
ORDER BY count(p) DESC
LIMIT 10;
</pre>

This gives a more reasonable list:

<pre>
+------------------------------------------------------------------------+
| p.id                                                        | count(p) |
+------------------------------------------------------------------------+
| "snowplowanalytics.com/about/index.html"                    | 491      |
| "snowplowanalytics.com/product/index.html"                  | 358      |
| "snowplowanalytics.com/services/index.html"                 | 345      |
| "snowplowanalytics.com/"                                    | 260      |
| "snowplowanalytics.com/technology/index.html"               | 256      |
| "snowplowanalytics.com/analytics/index.html"                | 251      |
| "snowplowanalytics.com/about/users.html"                    | 211      |
| "snowplowanalytics.com/about/community.html"                | 146      |
| "snowplowanalytics.com/product/get-started.html"            | 112      |
| "snowplowanalytics.com/product/do-more-with-your-data.html" | 97       |
+------------------------------------------------------------------------+
</pre>

The <tt>[:PREV*2]</tt> means we can easily amend the query to find the page users were on, for example, 5 steps before the 'About' page:

<pre>
MATCH path=(about:Page {id:"snowplowanalytics.com/about/index.html"})<-[:OBJECT]-(c_view)<-[:PREV*5]-(prev2)-[:OBJECT]->(p:Page) 
WHERE none(
	v IN NODES(path)[2..LENGTH(path)-1]
	WHERE v.page = about.id
)
RETURN p.id, count(p)
ORDER BY count(p) DESC
LIMIT 10;
</pre>

This is the kind of search that would be difficult in SQL because we'd have to either sort the list or perform a lot of searches. Noe4J takes 450ms to give us:

<pre>
+-------------------------------------------------------------------------------+
| p.id                                                               | count(p) |
+-------------------------------------------------------------------------------+
| "snowplowanalytics.com/analytics/index.html"                       | 130      |
| "snowplowanalytics.com/product/index.html"                         | 122      |
| "snowplowanalytics.com/technology/index.html"                      | 118      |
| "snowplowanalytics.com/services/index.html"                        | 108      |
| "snowplowanalytics.com/about/index.html"                           | 82       |
| "snowplowanalytics.com/"                                           | 81       |
| "snowplowanalytics.com/product/get-started.html"                   | 50       |
| "snowplowanalytics.com/blog.html"                                  | 40       |
| "snowplowanalytics.com/product/do-more-with-your-data.html"        | 39       |
| "snowplowanalytics.com/analytics/customer-analytics/overview.html" | 38       |
+-------------------------------------------------------------------------------+
</pre>

Notice that the numbers have reduced significantly. That's because we're looking for paths 5 pages long (or specifically 5 events long) that don't include the 'About' page. That might be because users tend to find that page in fewer than 5 steps, and they may then leave it but come back to it again within 5 steps.

Again, we can visualise a couple of these paths in the browser console:

<p style="text-align:center"><img src="/assets/img/blog/2014/07/Neo4j-5-step-path.png"></p>

It's worth considering at this point how long it would take to perform an equivalent search in SQL. For each session by each user, we'd have to find all of the page view events, sort them, find the times they've visited the 'About' page, and then count five steps back, checking that the intervening pages don't include the 'About' page.










## What journeys do users take from the home page?

We might be interested in where users go after arriving at the homepage. Now we'll need to match a pattern of, say, 3 steps from the homepage. We'll use the <tt>EXTRACT</tt> command to return just the URL attached to the events in the path, rather than the nodes themselves. That's because we're not looking for user IDs, timestamps, etc, so this will give us some cleaner results.

<pre>
MATCH path = (home:Page {id: "snowplowanalytics.com/"})<-[:OBJECT]-(home_view)<-[:PREV*3]-(:View)
RETURN EXTRACT(v in NODES(path)[2..LENGTH(path)+1] | v.page), count(path)
ORDER BY count(path) DESC
LIMIT 10;
</pre>

This took 696 ms to return the 10 most common paths from the homepage. I've removed 'snowplowanalytics.com' from each URL to save space. 

<pre>
+----------------------------------------------------------------------------------------------------------------------------+
| ["/product/index.html","/services/index.html","/analytics/index.html"]                                       | 1627        |
| ["/","/","/"]                                                                                                | 1119        |
| ["/product/index.html","/product/do-more-with-your-data.html","/analytics/customer-analytics/overview.html"] | 211         |
| ["/product/index.html","/product/do-more-with-your-data.html","/product/get-started.html"]                   | 192         |
| ["/","/product/index.html","/services/index.html"]                                                           | 181         |
| ["/services/index.html","/analytics/index.html","/technology/index.html"]                                    | 178         |
| ["/product/index.html","/services/index.html","/product/index.html"]                                         | 165         |
| ["/product/index.html","/services/index.html","/services/reporting.html"]                                    | 138         |
| ["/product/index.html","/analytics/index.html","/technology/index.html"]                                     | 129         |
| ["/product/index.html","/product/do-more-with-your-data.html","/services/index.html"]                        | 121         |
+----------------------------------------------------------------------------------------------------------------------------+
</pre>

Notice that the most common route is to click, from left to right, on each of the titles listed along the top of the Snowplow website. Notice also that the second most common path is just repeatedly refreshing the home page. We could modify the search to exclude these using a WHERE clause as we did in the last example.















## What are the most common journeys that end on a particular page?

This time we'll look at paths that lead to the 'About' page. The only changes we need to make from our previous example is to change the target page and reverse the *prev* edges. But just to keep things varied, let's exclude paths that include the 'About' page before the end.

<pre>
MATCH path = (about:Page {id: "snowplowanalytics.com/about/index.html"})<-[:OBJECT]-(home_view)-[:PREV*3]->(:View)
WHERE none(
	v IN NODES(path)[2..LENGTH(path)+1]
	WHERE v.page = about.id
)
RETURN EXTRACT(v in NODES(path)[2..LENGTH(path)+1] | v.page), count(path)
ORDER BY count(path) DESC
LIMIT 10;
</pre>

This takes only 443 ms to give these results (which are backwards - the first page in the journey is on the right, and the left-most page is the page they were on immediately before arriving at the 'About' page):

<pre>
+----------------------------------------------------------------------------------------------------------+
| ["/blog.html","/technology/index.html","/analytics/index.html"]                            | 155         |
| ["/blog/index.html","/technology/index.html","/analytics/index.html"]                      | 133         |
| ["/technology/index.html","/analytics/index.html","/services/index.html"]                  | 128         |
| ["/analytics/index.html","/services/index.html","/product/index.html"]                     | 46          |
| ["/","/","/"]                                                                              | 36          |
| ["/services/index.html","/product/index.html","/"]                                         | 29          |
| ["/product/index.html","/services/index.html","/analytics/index.html"]                     | 25          |
| ["/","/product/index.html","/services/index.html"]                                         | 16          |
| ["/","/product/index.html","/"]                                                            | 16          |
| ["/product/get-started.html","/product/do-more-with-your-data.html","/product/index.html"] | 16          |
+----------------------------------------------------------------------------------------------------------+
</pre>










## How long does it take users to get from one page to another?

In order to understand how users are using a website, we may want to measure how long it took them to get from a specified page to another specified page, measured in terms of the number of intermediate pages. We can do that in Neo4J by first matching the pages we're interested in as well as the pattern joining them. 

Then we want to exclude journeys that have either the start or end page as intermediate steps. There are two good reasons for doing this. Consider a user who arrives at the home page, reads some of the pages in the 'Services' section of the site, and then returns to the home page and goes directly to the blog. According to our matching rules, this user would be counted twice: once from his first visit to the home page, and again for his second visit. And it seems reasonable to rule out the longer journey: after all, it seems that they weren't looking for the blog when they first arrived at the home page.


<pre>
MATCH (blog:Page {id:"snowplowanalytics.com/blog/index.html"}),
(home:Page {id:"snowplowanalytics.com/"}),
p = (home)<-[:OBJECT]-()<-[:PREV*..10]-()-[:OBJECT]->(blog)
WHERE NONE(
	v IN NODES(p)[2..LENGTH(p)-1]
	WHERE v.page = blog.id
	OR v.page = home.id
)
RETURN length(p), count(length(p))
ORDER BY length(p)
</pre>

This takes around 9 seconds to return this table. Note that the lengths measure the number of *edges* in the path. We don't want to include the two *object* edges, so we should subtract 2 to find the number of 'clicks' between the home page and blog index, or 3 to find the number of intermediate pages.

+------------------------------+
| length(p) | count(length(p)) |
+------------------------------+
| 3         | 482              |
| 4         | 183              |
| 5         | 120              |
| 6         | 124              |
| 7         | 233              |
| 8         | 74               |
| 9         | 49               |
| 10        | 46               |
| 11        | 25               |
| 12        | 22               |
+------------------------------+

The bump at length = 7 (five clicks) is probably due to the site architecture: 'Blog' is the fifth link along the top of the website.

Again we can visualise some of these journeys in the browser console. Neo4J lets us click on the view nodes to see the page they're associated with.

<p style="text-align:center"><a href="/assets/img/blog/2014/07/Neo4j-paths-between-home-and-blog.png"><img src="/assets/img/blog/2014/07/Neo4j-paths-between-home-and-blog.png"></a></p>
