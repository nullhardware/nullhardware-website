---
title: Wireless Smart Sensors for BetaCityYeg
headline: Cities use Open Hardware to Get (Cheap) Open Data
description: >-
  Marcin's latest embedded projects are wireless sensors for open data
  collection, part of why the City of Edmonton leads the way when it comes to
  opendata.
date: 2017-06-15T17:15:35.057Z
author: 'marcin, andrew'
draft_img: /img/drafts/betacity-board-stack-00000.jpg
tags:
  - opendata
excerpt: >-
  Cities are put in a tough spot. They want to provide their citizens with the
  best possible benefits, the best possible transportation, and the best
  possible reasons to live there. However, they don't really have the data they
  need to make informed decisions. BetaCityYEG has a vision of developing open
  civic technology to make this possible. After all, how can you serve your
  citizens without understanding them first?
---
Cities are put in a tough spot. They *want* to provide their citizens with the best possible benefits, the best possible transportation, and the best possible reasons to live there. However, they don't really have the data they need to make informed decisions. [BetaCityYEG](https://betacity.ca/) has a vision of developing open civic technology to make this possible.

> After all, how can you serve your citizens without understanding them first?

## Open data and Citizen Science

For over a year now, I've been working with [BetaCityYEG](https://betacity.ca/) ([www.betacity.ca](https://www.betacity.ca)) designing hardware to help the [City of Edmonton](http://www.edmonton.ca) collect [data](https://data.edmonton.ca/). The concept is to use open-source platforms (Arduino, Raspberry Pi, etc...), tie them in to sensors, and give the city some much needed insight into the inner workings of Edmonton.

Edmonton is actually a surprising spot for an initiative like this. We have long, dark winters, and the extreme cold is hell on electronics. It isn't an easy task to take off-the-shelf modules and turn them into a platform rugged enough to survive out here. But the payoff is a something that will eventually affect the lives of everyone who lives here.

The implications of arming a city (and its citizens) with data was discussed in a recent Edmonton Journal [article](http://edmontonjournal.com/news/local-news/edmontons-new-smart-city-data-network-aims-to-arm-citizens-with-the-facts). Projects like this clearly have the potential to transform civic debate and planning and push data-driven decision making, which is exactly why BetaCityYEG is dedicated to developing low-cost citizen-led open-data initiatives.

## Unglamorous and Ugly

I presented some of my work at the recent  [Canadian Open Data Summit](http://opendatasummit.ca/). The truth is that to the layperson, these projects are unsexy. They don't contain advanced electronics, and if you saw one, you probably wouldn't even give it a second glance. The real value of these projects is combining what appear to be simple data points into a living, breathing, dynamic model of exactly the City of Edmonton is up to.  While the City does have resources to do focused studies, the red tape and lack of resources does impact it’s ability to respond quickly and effectively to citizen’s concerns.  These projects shift the balance of power back to the citizens, allowing them to voice their concerns and back them up with real data.

Over the last couple years there has been heavy investment in the downtown core, particularly around the newly opened [Rogers Place](http://www.rogersplace.com/), home of the [Edmonton Oilers](https://www.nhl.com/oilers). As a result, the [City of Edmonton](http://edmonton.ca) has been very interested in how traffic and pedestrian patterns may have changed. My first project was the hardware to help quantify the flow of pedestrians in this newly revitalized downtown district. *What does it look like?* It's an unassuming box with a motion sensor.

![Open Source Pedestrian Counter](/img/drafts/ped-counter-box-00000.jpg)

Behind the scenes, it presents the data it logs in an easy to understand graph, while making the raw data available for detailed analysis. Basically, each sensor unit contains a protected WiFi access point. You login to the access point with your password, you connect to the web interface, and you download the most recent data with a click.  This early version used WIFI as a stand-in for the LoRA network connectivity that is in the works, and while requiring someone to go and collect the data, it is “good enough” to get the ball rolling and get data flowing back to the interested parties.

Here's a peak into an early development version of that interface:

![Graph of pedestrian traffic in Edmonton during trial.](/img/drafts/pedestrain-counter-graph-00000.jpg)

## Weather Monitoring

![Weather Statio Build](/img/drafts/weather-station-build-00000.jpg)

[Air quality monitoring is extremely important](http://capitalairshed.ca/news/new-efforts-aim-to-understand-poor-air-quality-in-edmonton) for northern cities like Edmonton who have a large base of industrial businesses in close proximity to populated areas.    In fact, your neighborhood most likely faces conditions which are totally different from your friend across town. Unfortunately, the official air quality testing done by the Province of Alberta is [limited to a few sites](http://edmontonjournal.com/news/local-news/new-air-quality-gadgets-demystify-pollution).  As a result, BetaCityYEG was tasked with the development of a cheap and easily deployable mechanism for measuring air quality plus other important factors such as wind speed, rainfall, temperature and humidity. The availability of more granular data on a neighbourhood level is critical for better serving the citizens of Edmonton, as well as planing and zoneing future development.  Preliminary results were presented at the Canadian Open Data Summit, and when completed, plans are to release the entire design as an open-source project.

## A connected future

In the near future, the hope is that all of these related projects will communicate with each other and all of the data will centralized into one convenient source. This is the goal of city-wide network technologies such as LoRa/LoRaWAN™. Instead of using an off-the-shelf Wifi or Bluetooth module, the hope is to add wireless modules that are capable of communicating over several kilometers. Being able to collect data from all over Edmonton is a goal of BetaCityYEG, and it's one that will provide immediate value, given the distributed layout of the City of Edmonton.

## Coming Up

BetaCityYEG and the City of Edmonton are both dedicated to improving [#opendata](https://twitter.com/search?q=opendata). In recognition of this leadership, the city was named Canada's Most Open City in June 2017, for the second year in a row. For Edmonton, continued investment in open hardware and other open data initiatives are absolutely a must. I am very excited to be a part of these initiatives, and I hope to help provide more open hardware and open data to the City.

If **you** have an idea or a project that can help cities better serve their citizens, we'd love to hear about it. Please [send us an email](mailto:admin@nullhardware.com).
