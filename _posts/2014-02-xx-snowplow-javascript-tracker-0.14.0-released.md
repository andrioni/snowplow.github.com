---
layout: blog-post
shortenedlink: JavaScript Tracker 0.14.0 released
title: Snowplow JavaScript Tracker 0.14.0 released with new features
tags: snowplow javascript tracker
author: Fred
category: Releases
---

We are happy to announce the release of the [SnowPlow JavaScript Tracker version 0.14.0] [0140-release]. 

In this release we have introduced some new tracking options and compressed our tracker for better load times. We have also updated our build process to use [Grunt] [grunt].

This blog post will cover the following changes:

1. [New feature: gzipping]
2. [New feature: set user ID from a cookie]
3. [New feature: set user ID from a querystring]
4. [New feature: pass a referrer through an iframe]
5. [New feature: respecting Do Not Track]
6. [New build process]
7. [Other structural improvements]
8. [Upgrading]
9. [Getting help]

<h2><a name="gzipping">1. New feature: gzipping</a></h2>

Our hosted JavaScript JavaScript Tracker is now gzipped for CloudFront delivery using a Grunt plugin. This decreases the file size by about 70%, improving load times and perceived website speed. All modern browsers support gzipping ([reference] [gzip-browsers]).

<h2><a name="cookie">2. New feature: set user ID from a cookie</a></h2>

This new function allows you to set the visitor's business user ID to the value of a first-party cookie. Its signature is:

{% highlight javascript %}
function setUserIdFromCookie(cookieName);
{% endhighlight %}

For example, this code:

{% highlight javascript %}
_snaq.push(['setUserIdFromCookie', '_sp_id.4209']);
{% endhighlight %}

will look for a cookie whose name is "_sp_id.4209". If one is found, the user ID will be set to the value of that cookie.

<h2><a name="querystring">3. New feature: set user ID from a querystring</a></h2>

We have added support for setting the visitor's business user ID from a field in the querystring of either the current page's URL or the referring page's URL.
The signatures for the new functions are:

{% highlight javascript %}
function setUserIdFromLocation(querystringField);
function setUserIdFromReferrer(querystringField);
{% endhighlight %}

For example, to set the user's ID to the value of the "id" field in the local querystring, use:

{% highlight javascript %}
_snaq.push([['setUserIdFromLocation', 'id']]);
{% endhighlight %}

And to set it to the value of the "id" field in the referrer querystring, use:

{% highlight javascript %}
_snaq.push([['setUserIdFromReferrer', 'id']]);
{% endhighlight %}

<h2><a name="iframe">4. New feature: pass a referrer through an iframe</a></h2>

You can now pass the referrer back to Snowplow from inside an iframe. To do this, set `referrer=x` in the querystring of the iframe's URL, where 'x' is the required referrer. This also works if you set `referer=x` (with one 'r').

<h2><a name="iframe">5. New feature: respecting Do Not Track</a></h2>

Section to add.

<h2><a name="grunt">6. Switched to Grunt-based build process</a></h2>

We have replaced our custom `snowpak.sh` Bash script with a standardised [Grunt] [grunt] file. This let us use Grunt plugins to automate the build process, from file concatentation and YUI Compression through to uploading to S3 and CloudFront invalidation.

It should also help us move to a more modular project structure and add a test suite in the [next release] [100-issues].

<h2><a name="housekeeping">7. Other structural improvements</a></h2>

We have also:

* Moved cookie-related functionality into a new file [(#77)] [https://github.com/snowplow/snowplow-javascript-tracker/issues/77]
* Moved requestStringBuilder into a new file [(#55)] [https://github.com/snowplow/snowplow-javascript-tracker/issues/55]
* Moved functions to detect browser attributes into a new file [(#37)] [https://github.com/snowplow/snowplow-javascript-tracker/issues/37]
* Renamed setDoNotTrack to respectDoNotTrack [(#28)] [https://github.com/snowplow/snowplow-javascript-tracker/issues/28]
* Removed getLegacyCookieName [(#50)] [https://github.com/snowplow/snowplow-javascript-tracker/issues/50]
* Removed the legacy Piwik debug code [(#65)] [https://github.com/snowplow/snowplow-javascript-tracker/issues/65]

<h2><a name="upgrading">8. Upgrading </a></h2>

The upgraded minified tracker is available here:

    http(s)://d1fc8wv8zag5ca.cloudfront.net/0.14.0/sp.js

If you are currently using the path:

    http(s)://d1fc8wv8zag5ca.cloudfront.net/0/sp.js

then you will get the new version automatically.

<h2><a name="help">9. Getting help </a></h2>

Check out the [v0.14.0 release page] [0140-release] on GitHub for the full list of changes made in this version.

As always, if you run into any issues or don't understand any of the above changes, please [raise an issue] [issues] or get in touch with us via [the usual channels] [talk-to-us].

[gzip-browsers]: http://webmasters.stackexchange.com/questions/22217/which-browsers-handle-content-encoding-gzip-and-which-of-them-has-any-special

[0140-release]: https://github.com/snowplow/snowplow-javascript-tracker/releases/tag/0.14.0
[100-issues]: https://github.com/snowplow/snowplow-javascript-tracker/issues?milestone=4&page=1&state=open

[grunt]: [http://gruntjs.com/]

[issues]: https://github.com/snowplow/snowplow/issues
[talk-to-us]: https://github.com/snowplow/snowplow/wiki/Talk-to-us