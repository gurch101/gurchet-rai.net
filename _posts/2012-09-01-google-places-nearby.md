---
layout: post
title: Using the Google Places API 
summary: Finding Nearby Points of Interest with the Google Places API
date: 2012-09-01
tags: javascript project 
---

Demo @ [http://www.gurchet-rai.net/apps/places/](http://www.gurchet-rai.net/apps/places/)

The Google Places API is a web service that returns yelp-like information (ratings, reviews, contact info) based on your location. Using the HTML5 geolocation API described in my [last post](http://www.gurchet-rai.net/html5-geolocation-api), I’ll go over how I made a simple little app that lets you view details for places near you.

### Getting a Map Near You

The Google Maps V3 Javascript API can be used to easily get a map centered around your location.

First, include the maps API and the places API:

{% highlight javascript %}
<script type="text/javascript" src="http://maps.googleapis.com/maps/api/js?libraries=places&sensor=false"></script>
{% endhighlight %}

Using the coordinates from the geolocation API (described [here](http://www.gurchet-rai.net/dev/html5-geolocation-api)), construct a map as follows:

{% highlight javascript %}
var loc = new google.maps.LatLng(coords.latitude, coords.longitude);
// map_canvas must have a defined height and width
var map = new google.maps.Map(document.getElementById("map_canvas"), {
              mapTypeId: google.maps.MapTypeId.ROADMAP,
              center: loc,
              zoom: 13
});
{% endhighlight %}

### Getting Local Points of Interest

Now that we have a map, we can integrate the Google Places API and build a localized search. The following will get up to 20 results within 5 km of the maps center point (paging is ignored for this example).

{% highlight javascript %}
var resultList = [];
var service = new google.maps.places.PlacesService(map);  
var request = {
    location: map.getCenter(),
    radius: '5000',
    query: <search string>            
};

service.textSearch(request, function(results, status, pagination){
    if (status == google.maps.places.PlacesServiceStatus.OK) {
        resultList = resultList.concat(results);
        plotResultList();
    }
});
{% endhighlight %}

### Plotting the Result List

To plot the result list from above, I wrote the following function to drop marker overlays.

{% highlight javascript %}
var overlays = [];
        
function plotResultList(){
    var iterator = 0;                  
    for(var i = 0; i < resultList.length; i++){
        setTimeout(function(){
                    
            var marker = new google.maps.Marker({
                position: resultList[iterator].geometry.position,
                map: map,
                title: resultList[iterator].name,
                animation: google.maps.Animation.DROP
            });

            overlays.push(marker);
                    
            iterator++;
        }, i * 75);
    }
}
{% endhighlight %}

### Show Additional Details for each Place

The Google Places API can be used to fetch information like contact info, ratings, and reviews. Here’s some code to overlay some of this info when a marker is clicked.

{% highlight javascript %}
var infoWindow = new google.maps.InfoWindow();      
google.maps.event.addListener(marker, 'click', function() {
    infoWindow.close();
    var request = {
        reference: resultList[iterator].reference
    };

    service.getDetails(request, function(place, status){
        var content = "<h2>" + name + "</h2>";
        if(status == google.maps.places.PlacesServiceStatus.OK){    
            if(typeof place.rating !== 'undefined'){
                content += "<p><small>Rating: " + place.rating + "</small></p>"; 
            }    
            
            if(typeof place.formatted_address !== 'undefined'){
                content += "<br><small>" + place.formatted_address + "</small>";
            }
            
            if(typeof place.formatted_phone_number !== 'undefined'){
                content += "<br><small><a href='tel:" + place.formatted_phone_number + "'>" + place.formatted_phone_number + "</a></small>";                                 
            }
            
            if(typeof place.website !== 'undefined'){
                content += "<br><small><a href='" + place.website + "'>website</a></small>";
            
            }
        }                            
        
        infoWindow.setContent(content);
        infoWindow.open(map, marker);         
    });
});
{% endhighlight %}
