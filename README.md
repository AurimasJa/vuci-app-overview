# Vuci overview application
This application is created for my personal learning.
The project involves creating a Vuci application that focuses upon router overview information including draggable cards.
## Uci configuration
Using uci configuration (overview) manipulating data.
```
config card
  option position 0
  option isVisible true
  option title 'title'
  option id 0
```
It will be determined how cards will be displayed based on position.
## Vuci overview
Show all available network, system, and log information. Cards must be draggable, and positions must be changed. There is a drawer in which we can hide cards.

## Overview folder
This folder contains all logic that will be displayed in System/Overview menu.

## Interfaces folder
Modified exsisting code to add and remove UCI configuration on network add/remove


## Photos
System information displayed in cards:
![MicrosoftTeams-image](https://github.com/AurimasJa/vuci-app-overview/assets/79587555/cce5f553-b550-4eff-a9b4-bf846a883645)

Drawer that can hide or display cards:
![MicrosoftTeams-image (2)](https://github.com/AurimasJa/vuci-app-overview/assets/79587555/56c2a070-94ec-4ed5-a3e4-28b376924d83)
