---
title: Wireless Smart Sensors for BetaCityYeg
headline: Cities use Open Hardware to Get (Cheap) Open Data
description: >-
  Marcin's latest embedded projects are wireless sensors for open data
  collection, part of why the City of Edmonton leads the way when it comes to
  opendata.
date: 2017-06-15T17:15:35.057Z
author: marcin
draft_img: /img/drafts/betacity-board-stack-00000.jpg
tags:
  - opendata
excerpt: >-
  doh. not actually optional at the moment. Put the text that you want to appear
  in the "blog post listing" here.
---
The problem is that cities today want to have it both ways. They want to provide their citizens with the best possible benefits, the best possible transportation, and the best possible reasons to live there. However, they don't always understand the intricate details about the daily lives of their own citizens.

How do you serve your citizens without understanding them first?

## Open data

For over a year now, I've been helping the City of Edmonton collect data to better understand it's constituents. In fact, the [city](http://www.edmonton.ca) has once again been named Canada's Most Open City in the 2017 Open Cities Index, in part due to the dedication of [BetaCityYeg](https://betacity.ca/).

It has been a true honor to help develop *Citizen Science* initiatives like the *Pedestrian Counter* (now in version 3.0) and a *Weather Station* for air quality monitoring here in Edmonton.

## Citizen Science

Part of the mission of [BetaCityYEG](https://betacity.ca/) was recently discussed in an article in the [Edmonton Journal](http://edmontonjournal.com/news/local-news/edmontons-new-smart-city-data-network-aims-to-arm-citizens-with-the-facts).  The idea is to create a low-cost citizen-lead  initiative that could provide the [City of Edmonton](http://edmonton.ca) with the data it needs to better serve it's constituents. For over the last year, I've been helping develop open hardware (and software) to better help the city make informed decisions.

Our fist project was a Pedestrian Counter, which was aimed at quantifying the flow of pedestrians in the newly revitalized downtown district. As many of you may know, there has been a critical investment in the downtown area over the past couple of years, including the newly open [Rogers Place](http://www.rogersplace.com/), home of the [Edmonton Oilers](https://www.nhl.com/oilers). As a result, the [City of Edmonton](http://edmonton.ca) has been very interested in how traffic and pedestrian patterns may have changed. BetaCityYEG is part of a citizen lead movement to better understand how investment in infrastructure might change the behavior of Edmonton citizens. To do that, they have turned to measuring pedestrian traffic in critical areas, such as those immediately surrounding Rogers Place.

As a first step, a simple meter was developed to estimate pedestrian traffic.

![Graph of pedestrian traffic in Edmonton during trial.](/img/drafts/pedestrain-counter-graph-00000.jpg)

To simply the collection of data for everyone involved, open hardware was used to collect the data using available sensors, and present it in an easy to understand format that could be collected by authorized representatives of the City of Edmonton. For anyone who is interested in the technical details, each sensor unit contained a protected WiFi access point that provides a web interface (including a graph and a download link for the most recently collected data). Authorized representatives are able to collect the data by connecting to the access points and downloading the most recent data.

## Weather Monitoring

Air quality monitoring is extremely important for cities like Edmonton who have a large base of industrial businesses in close proximity to populated areas. In fact, individual neighborhoods may have their own unique needs and/or concerns regarding air quality. Unfortunately, the official air quality testing done by the Province of Alberta isn't detailed enough for the city to make informed decisions. As a result, BetaCityYEG was tasked with the development of mechanism for measuring air quality and other important factors such as wind speed, rainfall, temperature and humidity. This data is critical for better serving the citizens of Edmonton, as well as planing and zoneing future development.

## A connected future

Although these projects are exciting, they represent independent projects with their own agendas. In the near future, the hope is that all of these related projects will communicate with each other and all of the data will centralized into one convenient source. This is the goal of city-wide network technologies such as LoRa/LoRaWAN™. Instead of using an off-the-shelf Wifi or Bluetooth module, the hope is to add wireless modules that are capable of communicating over several kilometers. Being able to collect data from all over Edmonton is a goal of BetaCityYEG, and it's one that will provide immediate value, given the distributed layout of the City of Edmonton. 

## Coming Up

BetaCityYEG and the City of Edmonton are both dedicated to improving [#opendata](https://twitter.com/search?q=opendata) and maintaining Edmonton's leadership as Canada's most Open City. To do that, investment in open hardware and other open data initiatives are absolutely a must. I am very excited to be a part of these initiatives, and I hope to help provide more open hardware and open data to the City.

If you have an idea or a project that can help cities better serve their citizens, we'd love to hear about it. Please [let us know](mailto:admin@nullhardware.com).

