---
layout: post
title: "Pokemon IRL"
author: "James Finlay"
author-img: james_finlay.png
date: 2014-05-07 13:07:43 -0600
comments: true
categories: [GIS, Nodejs, Android, Game]
---

Last summer I implemented a location-based game for Android during my spare time. The game mimics the exploration and training aspects of Pokemon, but in the real world. This post explains how the project is implemented, as well as a view on the back-end technologies.

<!-- more -->

For any who haven't played the Pokemon games, the gameplay consists of exploring a world of monsters. If your character is in a forest, you find leaf creatures. In water, you find water monsters. You capture these monsters and train them by fighting against other trainers. Google recently made an [April Fools Day challenge](http://features.peta.org/pokemon-black-and-white-parody/) where you try to find Pokemon at static locations on Google Maps. As well, PETA released their own [parody some time ago](http://features.peta.org/pokemon-black-and-white-parody/), though they clearly take a different stance towards it.

My goal was to make a mobile geolocation app that mimics the exploration aspect of the original gameplay, except in the real world. The app takes your geographic location, maps it against nearby features (forest, river, etc), and then generates a creature encounter. The server implementation was successful for the map data I gathered (Alberta and Ontario), though I grew bored of implementing the front end battle code and moved on to other projects.

This post is about the technology and implementation of the geolocation service and its integration with mobile.

## ArcGIS

[ArcGIS](http://www.arcgis.com) is a mapping platform for Geographic Information Systems (GIS). I learned it originally through my work term at the provincial government. GIS platforms provide a similar service to Google Maps, where you interact with digital maps, but they provide much more depth and control. Instead of having a just shape imagery or satellite imagery, such as you would have on Google Maps or Bing, you can overlay as many different layers as you want. There are layers with foresty information, landmarks, mines, rivers, and more. 


{% img /images/posts/2014-04-11-pokemon-irl/google_maps.jpg %}


GIS platforms can also provide 3D modelling tools and more advanced plug-ins, but this is out of scope of the project.

There are a variety of different (and cheaper) [alternatives](http://en.wikipedia.org/wiki/List_of_geographic_information_systems_software#Notable_commercial_or_proprietary_GIS_software) to ArcGIS, though Esri's service provides a lot of useful features, foremost being .NET libraries and an integrated Python console. I found a few open source GIS programs, though I haven't had the opportunity to work with them.

To make the geolocation queries fast during runtime, I used ArcGIS' Python package to merge all used layers into a single shapefile. From the result I perform a quick location query to see what feature was hit. With the data all in the proper projections, the API is very easy to work with -- though building the formatted map is slow when intersecting a whole province. 

{% coderay lang:python %}
# Cursors
bufferedFeatures = arcpy.SearchCursor(temp_features)
targetFeatures = arcpy.SearchCursor(out_features);
insertCursor = arcpy.InsertCursor(out_features);

for feature in bufferedFeatures:
	# format the output geometry
	theShape = feature.getValue("Shape")

	# take difference to ensure no overlap
	# this is obviously the slow / poorly optimized part
	for ofeature in targetFeatures:
		oShape = ofeature.getValue("Shape")
		if theShape.intersect(oShape, 4):
			theShape = theShape.difference(oShape);

    # add as new feature
    newFeature = insertCursor.newRow()
    newFeature.type = spatial_type
    newFeature.source = env.workspace + in_features
    insertCursor.insertRow(newFeature)
{% endcoderay %}

It is always interesting to see how grade school math turns out useful, even if it is just basic geometry.

## Nodejs & PostGIS

So the maps can now be easily queried, but we need an easy way to actually perform this query. Requirements being that

1.	The system has an Android implementation (simple)
2.	Network connection is minimal, as is the query time (still simple)
3.	I can use my old laptop that I accidentally ran over with a car as the server because I am too cheap to bother with a better solution (bit more difficult)

What made the most sense was an integration of [Nodejs](http://nodejs.org/) and [PostGIS](http://postgis.net/).

PostGIS is a special extension for PostgreSQL that provides spatial services. It can perform spatial relationships, measurements, geometry, and a ton of other nifty functions. The project's implementation only used some basic methodology, but it is a fantastic open-source resource.

By integrating with a Nodejs, the server was incredibly easy to implement. When the mobile device wants to trigger a creature encounter, it uses the GPS to pull the latitude and longitude. The phone sends these values to the NodeJS server, which uses PostGIS to query for intercepting features.

{% coderay lang:javascript %}
// the query
var theQuery = "SELECT objectid, geom, ST_Distance(ST_Transform(ST_GeomFromText('POINT(" + data.longitude + " " + data.latitude + ")', 4326), 3400), geom) As dist FROM watercourses ORDER BY dist ASC LIMIT 1;";
var query = spatial_client.query(theQuery);

//can stream row results back 1 at a time
query.on('row', function(result) {
	console.log("Watercourse within " + result.dist + " metres");

	if (result.dist < 100)
		client.emit("result", {value: "w"});
	else
		client.emit("result", {value: "g"});
});

// on query completion, close the connection
query.on('end', function() {
	console.log("Query ended");
	spatial_client.end();
});
{% endcoderay %}

Only 'w' (water) or 'g' (grass) are returned in the example. This can be expanded to work with any geographic data available; such as suburban areas, provincial parks, cemeteries, mountains, Burger King restaurants, etc. Dependent on the data returned, the phone can then throw a creature encounter.

## GIS Data

I found the most difficult part of this project to be gathering the map data. I found a number of resources available with decent shapefiles, but it is all dependent on user contributions. This creates a lot of differing standards, concentration on cities, and inconsistent data. The provincial government has some great data that would have been extremely useful, but I wasn't allowed to use it (even after asking politely).

Some great sources I did find are:

- geogratis.gc.ca
- asterweb.jpl.nasa.gov/gdem.asp
- grasswiki.osgeo.org/wiki/Global_datasets
- srtm
- reverb.echo.nasa.gov

And all the various contributors at OpenStreetMap.org must be commended for their hard work. I added geographic data of my home town as recompense for the open source data I was using. Digitizing satellite imagery can be very relaxing.

## Open Weather

After working on some of the front end code for the app, I was bored and wanted to try integrating something new in the backend. If I am able to generate dynamic content from user location, why not also integrate that location's weather? On a sunny day, have flower-like monsters encounters. When it is raining, have more water Pokemon. And if there's a thunderstorm? Electric Pokemon.

After a quick search, I found a great REST API at [Open Weather Map](http://openweathermap.org/). I worked the API requests directly into the Android app and implemented a system to generate the creature encounter depending on the creature's rarity and the user's environment.

{% coderay lang:java %}
@Override
protected Weather doInBackground(Object... params) {
	...
	
	String address = "http://api.openweathermap.org/data/2.5/weather?" + "lat=" + latitude + "&lon=" + longitude;
	
	con = (HttpURLConnection) ( new java.net.URL(address)).openConnection();
	con.setRequestMethod("GET");
	con.setDoInput(true);
	con.setDoOutput(true);
	con.connect();

	...
{% endcoderay %}

I had the chance to test the weather data while in a thunderstorm last summer. However, it appeared the location weather station didn't report thunder so I didn't encounter any lightning monsters. I was a little upset.

---

I don't plan on talking about the Android implementation because it isn't quite as exciting and my game state logic is awful. However, if you'd like to see one of the more recent iterations of my code, it is available on [Github](https://github.com/Jtfinlay/GeoCritters). Feel free to use anything, though I'd recommend redoing the Android code from scratch. 