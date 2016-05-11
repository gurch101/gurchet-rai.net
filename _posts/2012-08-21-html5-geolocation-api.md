---
layout: post
title: Using the HTML5 Geolocation API 
summary: Getting a users position using the HTML5 Geolocation API 
date: 2012-08-21
tags: javascript html5
---

Demo @ [http://www.gurchet-rai.net/apps/geo/](http://www.gurchet-rai.net/apps/geo/)

I spent some time today fiddling around with the HTML5 Geolocation API for an upcoming project I’m working on and I thought I’d share some stuff I learned. In this post, I’ll briefly go over the API and the variability in accuracy I experienced when testing on a few different devices/browsers.

### Using the API

The Geolocation API is a high-level client-side API that returns a users longitude and latitude using the best available location information source (GPS, cell IDs, RFID, MAC address, Wi-Fi, bluetooth, or IP address). Here’s some code to get a users location:
{% highlight javascript %}
<script type="text/javascript">
try {
    if (navigator.geolocation) {
        /* 
            getCurrentPosition takes up to three arguments:
                a successCallback
                a errorCallback (optional)
                PositionOptions (optional)

            PositionOptions is an object with three properties:
                enableHighAccuracy (boolean - default is false
                                    if true, response times may be slowed
                                    and battery consumption (on mobiles) may 
                                    increase)
                timeout (in ms - default is infinity)
                maximumAge (in ms - default is 0
                            if > 0, the API tries to fetch a cached position)
        */
        navigator.geolocation.getCurrentPosition (function(position){
            /*
                the position object contains a 'coords' and 'timestamp' object;
                coords contains:
                    latitude
                    longitude
                    accuracy (in metres with 95% confidence)
                    altitude (in metres)
                    altitudeAccuracy (in metres)
                    heading (degrees clockwise relative to true north)
                    speed (in m/s - more relevant for the watchPosition function
                           which is not described here)
            */
        },
        function(error){
            /* 
                The error object has a 'code' and 'message' property
                error.code can be:
                    1 (permission denied), 
                    2 (position unavailable),
                    3 (timeout)         
            */      
        } 
    } else {
        /* browser doesn't support geolocation */   
    }
} catch (e) {
 /* error handling here */
}
</script>
{% endhighlight %}

### Error Handling
Several errors need to be handled when using the geolocation API.

A permission denied error (error code 1) may occur if the user denies the location lookup request. Note that this error will also be triggered if the user chooses ‘always deny’ on an alternate website making a geolocation request.

A position unavailable error (error code 2) may occur if the call to navigator.getCurrentPosition() fails. I was able to trigger this error on my phone by disabling Google Location Services.

Finally, the timeout error (error code 3) will occur if PositionOptions.timeout (if set) is exceeded.

### Accuracy

I experimented with the geolocation API on a few different browsers and devices. On my laptop, Chrome, Firefox, and Safari gave comparable results with an accuracy within 20 meters. IE, on the other hand, seemed to be be off by about 1.6 km. I was able to get down to within 5 meters on my cell phone (Galaxy Nexus) with GPS enabled.

### Integrating with Google Maps
For the purposes of this demo, I used the coordinates returned by the API to plot the users current location using the [Google Maps Static Maps API](https://developers.google.com/maps/documentation/static-maps/). You can see a demo of this [here](http://www.gurchet-rai.net/geo/index.html).
