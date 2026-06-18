# Should NYC Rideshare Drivers Accept Airport Dropoff Trips? A Regression and Simulation Analysis

## Overview 

Rideshare drivers widely regard airport dropoffs as lucrative due to longer distances and higher base fares, but the return trip uncertainty introduces real risk. Using 1.9 million NYC TLC High Volume FHV trips from 2024 to 2025, I built a regression model and Monte Carlo simulation to quantify whether airport trips actually pay more and when drivers should accept them. The regression finds that JFK, LGA, and EWR dropoffs carry statistically significant pay premiums of 21.5%, 22.0%, and 49.2% respectively over comparable city trips.
After dropping off at an airport, a driver must either join the virtual queue and wait for a return trip or deadhead back to the city empty. The true cost of an airport trip is therefore not just the fare but the opportunity cost of time spent waiting instead of completing city trips. However, simulation results show the airport advantage is time sensitive, with airport trips dominating across most boroughs and hours but the city baseline preferred during overnight hours (3 to 6am) when return trip probability is low.

## Motivation


