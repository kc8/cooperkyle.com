---

layout: post
title: Looking up Aircraft Data 
author: Kyle Cooper
category: blog 
excerpt_separator: <!--more-->

---

A lookup for local flight data collected via FlightAware or "Piaware"
<!--more-->

In a prior blog post I talked about a 'piaware' system that reads ADS-B data. However, this sytem does not easily allow you to translate to translate a given ICAO (aircraft identifier) into a registered tail number (often called the tail number).

I found a 'lookup table' for aircraft data that can help translate these numbers. I then build a small utility around that [here](https://github.com/kc8/get-aircraft-data#)

It works like this: download the publicly available registry data for all aircraft in the United States from the FAA's website: https://registry.faa.gov/database/ReleasableAircraft.zip
Then store this in a SQLite database inside a Docker container and make it available via a REST call. The docker container is important as anytime 
you want to refresh the table with new data, just restart the container and it rebuilds the database with the newly refreshed data.

From here you can get the tail number and the prefix for the number:
```
    curl "0.0.0.0:8080/icaoTranslate?icao=A03235"
    {"number":"111ZM","prefix":"N"}
```

Now I can quickly enrich the info from my piaware and store the enriched data having a history of the aircraft.
