---
layout: post
title: Getting All Articles Referenced in a Wikipedia Article 
summary: A bookmarklet to pull all wikipedia links from a wiki article.
date: 2013-05-26
tags: javascript
---
![Demo](./images/wiki.gif)

I wrote a little bookmarklet that lets you pull all Wikipedia article links referenced in any Wikipedia article. To try it out, create a bookmark with the following code as the link, navigate to a wiki article, and click on the bookmark.

{% highlight javascript %}
javascript:(function(){var e={isNavLink:function(e){return e.getAttribute("accesskey")!=null||e.parentNode.className.match(/icon/)!=null},isCitationLink:function(e){return e.parentNode.className.match(/citation/)!=null||e.parentNode.parentNode.className.match(/citation/)!=null},isWikiPageLink:function(e){return e.getAttribute("title")!=null&&e.getAttribute("href").indexOf("/wiki/")==0},isSpecialLink:function(e){return e.getAttribute("href").indexOf(":")>=0||e.getAttribute("style")!=null}};var t={};var n=document.links;var r=document.title;var i=document.createElement("h2");var s=document.createElement("div");var o='<table style="font-size: 16px; line-height:20px;">';o+='<thead style="font-weight:bold;"><tr><td>Referenced Articles</td></tr></thead><tbody>';for(var u=0;u<n.length;u++){var a=n[u];if(typeof t[a.href]==="undefined"){t[a.href]=true;if(e.isWikiPageLink(a)&&!e.isNavLink(a)&&!e.isSpecialLink(a)&&!e.isCitationLink(a)){var f=a.getAttribute("href");var l=a.getAttribute("title");o+='<tr><td><a href="'+f+'">'+l+"</a></td></tr>"}}}o+="</tbody></table>";i.innerHTML=r;s.innerHTML=o;document.body.innerHTML="";document.body.appendChild(i);document.body.appendChild(s);document.title=r})();
{% endhighlight %}

The unminified source can be found [here](https://github.com/gurch101/wikilinks).
