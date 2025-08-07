---

layout: post
title: Looking up Aircraft Data 
author: Kyle Cooper
category: blog 
excerpt_separator: <!--more-->

---

A lookup for local flight data collected via FlightAware or "Piaware"
<!--more-->

In a prior blog post I talked about a 'piaware' system that reads ADS-B data. 

An additional improvement I made to this setup was the ability to collect flights in long-term storage. I designed another system 
to do this (but that is not what this post is about). However, when getting the raw data for the aircraft, there 
was no way to translate the raw ID of the aircraft into registration (or its tail numbers). 

I found an interesting and cost-effective 'lookup table' for aircraft data So I built [one](https://github.com/kc8/get-aricraft-data#)

The idea of it is to download the publicly available registry data for all aircraft in the United States [from the FAAs website](https://registry.faa.gov/database/ReleasableAircraft.zip)
Then store this in  SQLite database inside a Docker container and make it available via a REST call. The docker container is important as anytime 
you want to refresh the table with new data, just restart the container and it rebuilds the database with the newly refreshed data.

From here you can get the tail number and the prefix for the number:
```
    curl "0.0.0.0:8080/icaoTranslate?icao=A03235"
    {"number":"111ZM","prefix":"N"}
```

Now I can quickly enrich the info from my piaware and store the enriched data having a history of the aircraft.
