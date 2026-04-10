# Welcome to the Open Data Hub Bootcamp 2026!
This repo will guide you through the challenge 1, in which you will visualize e-charging stations and their availability on a map.

The Data Visualization/Analysis topic foresees the creation of a visualization that integrates the following functionalities: 
create a simple view with list of e-charging stations and their availability
create a map visualization that shows: 
markers for all stations positioned according to their GPS coordinates 
color indicator of availability
extend the visualization to show a heatmap of availability
Extra: 
- show heatmap of usage frequency over time
- animate near real-time updates of stations


The goal of this bootcamp is to foster our community around the Open Data Hub, to get to know each other, the technology, but also for you to just learn stuff and make friends. This is not a competition, so don't be afraid to make mistakes and try out new stuff. Our Open Data Hub team is there to help and support you.

You will also have an opportunity to present the results to the public at our upcoming event, the Open Data Hub Day

# The challenge
Your challenge will be to implement a frontend application that displays e-charging stations and their availability on a map.

# Tech stack
You are free to use whatever programming language best fits your group, but since we're doing frontend work, javascript/typescript will likely be a good fit, and leaflet is an easy to use minimal library for this purpose

# Dataset
For this challenge you will use the e-charging dataset, which is part of the Open Data Hub's mobility domain.  
You can explore it using the
- [analytics frontend](https://analytics.opendatahub.com/) (under "E-Mobility")
- [API](https://mobility.api.opendatahub.com/v2/tree/EChargingStation/number-available/latest)
- [databrowser](https://databrowser.opendatahub.com/dataset-overview/6c9722ae-129f-47e9-92cc-0bde13a47ef3) 


## Stationtypes
The stations in the e-charging dataset are of type `EChargingStation` and `EChargingPlug`.  
How exactly these types are used depends on the specific provider, but generally:

### EChargingStation
Represents a location where one or more chargers are available. Their measurements show the number of available `plugs` at that location.  

### EChargingPlug
Represents a singular E-charging column. Plugs are children of a parent `EChargingStation`.  
Plugs usually have a measurement that shows if they are available or not.

Think of a parking area that has multiple charging spots. The whole parking area could be the `EChargingStation`, and the actual columns are be the `EChargingPlug`

## Data provider / origin

To guarantee data consistency, you should limit your dataset to one or more `origins`, or even stations (`scode`) by using the `where` filter. Stations from the same origin all have uniform metadata fields and data types.  
The largest and most reliable active provider of e-charging data is `Neogy` and we suggest using that for your challenge.

Use the analytics tool to find a station that is up to date, `active` and has a clean history.  

To access history of more than 100 days at a time, you have to register a user on analytics, see below ('Keycloak') for more information

# APIs
There are two APIs you will interact with:
- **Open Data Hub Time series API** [mobility.api.opendatahub.com](mobility.api.opendatahub.com)
 to retrieve the time series data both for training your algorithm and making the prediction
- **Keycloak** [auth.opendatahub.com/auth/](https://auth.opendatahub.com/auth/) to obtain an authorization token, needed for access to large time series datasets (optional, only for extra challenges)

Note that for this challenge you will interact with the `Time Series / Mobility` APIs, and not it's sibling, the `Content / Tourism` domain

## Time Series Objects and Concepts
Time series data takes the form of `Measurements` attached to `Stations`  

>[!NOTE]
>A measurement is a data point with a timestamp.

Each measurement has exactly one
- `mvalidtime`, the timestamp of the measurement
- `mvalue` the value of the measurement
- `mperiod`, the timeframe (in seconds) that the measurement references, and the periodicity with which it is updated. e.g. a temperature sensor that sends us it's data every 60 seconds has a period of 60.  
- `station`, a geographical point with an ID, name and some additional information. It's the location where measurements are made.
Fields referring to the station are prefixed with `s*`
- `data type`, which identifies what type of measurement it actually is. Is it a temperature in degrees Celsius? Is it the number of available cars? Is it the current occupancy of a parking lot?  
Fields referring to the data type are prefixed with `t*`

A **Station** might have multiple time series (list of measurements) of 0-n **Data types**, for example a weather station could have both `temperature` and `humidity` measurements.  
An e-charging station that we know exists, but doesn't provide any real time data, probably has no measurements at all.  
Stations may exist independently of measurements.
Stations have a `metadata` object, that contains additional information about the station.

This is a real world example of a parking station that has two data types, `free` and `occupied`, both with period 300. Some field's have been omitted for clarity's sake:
https://mobility.api.opendatahub.com/v2/tree/ParkingStation/*/latest?where=scode.eq.%22107%22,sactive.eq.true
```json
{
  "offset": 0,
  "data": {
    "ParkingStation": {
      "stations": {
        "107": {
          "sactive": true,
          "savailable": true,
          "scode": "107",
          "scoordinate": {
            "x": 11.351793,
            "y": 46.502958,
            "srid": 4326
          },
          "sdatatypes": {
            "free": {
              "tdescription": "free",
              "tmeasurements": [
                {
                  "mperiod": 300,
                  "mvalidtime": "2025-04-03 05:30:11.000+0000",
                  "mvalue": 144
                }
              ],
              "tname": "free",
              "ttype": "Instantaneous",
              "tunit": ""
            },
            "occupied": {
              "tdescription": "occupied",
              "tmeasurements": [
                {
                  "mperiod": 300,
                  "mvalidtime": "2025-04-03 05:30:11.000+0000",
                  "mvalue": 1
                }
              ],
              "tname": "occupied",
              "ttype": "Instantaneous",
              "tunit": ""
            }
          },
          "smetadata": {"capacity": 145, "municipality": "Bolzano - Bozen"},
          "sname": "P07 - Mareccio via C. de Medici",
          "sorigin": "FAMAS",
          "stype": "ParkingStation"
        }
      }
    }
  },
  "limit": 200
}
```
### Representation
All requests made against the Time Series API have to specify a `representation`, which is either `flat` or `tree`.  

`tree` displays the reponse in a structured tree format:
```
stationtype
  '─ station
    '─ datatype
      '─ [measurements]
```
`flat` flattens this tree structure, resulting in a list of measurements, where each measurement has it's station and datatype information repeated 

>[!TIP]
>Generally, `tree` is useful for initial exploration and when you need multiple data types grouped by station, while `flat` is better once you have tuned your `where` and `select` parameters to give you exactly the data you want.

### Latest vs history
The Api gives you two basic modes when interrogating measurements:  

- `/latest` only gives you the most recent measurement of each station. This is useful if you want to know the current state, e.g. if an Echarging station is occupied or not. Latest requests are specially optimized to be very fast
- `/<from>/<to>` lets you access a history between two dates, i.e. the actual time series. This is much slower than latest when querying large ranges and is subject to stricter quota limits (see section about keycloak)

## Keycloak
Keycloak is an Open Source Identity and Access management server (keycloak.org).
We use it to authenticate and authorize our services via the OAuth2 standard.
For you, this boils down to making a REST call supplying the credentials below, and you get back an access token.

```sh
curl -X POST -L "https://auth.opendatahub.com/auth/realms/noi/protocol/openid-connect/token" \
    --header 'Content-Type: application/x-www-form-urlencoded' \
    --data-urlencode 'grant_type=client_credentials' \
    --data-urlencode 'client_id=opendatahub-bootcamp-2025' \
    --data-urlencode 'client_secret=QiMsLjDpLi5ffjKRkI7eRgwOwNXoU9l1'
```

You then have to pass this token as `Authorization: Bearer <token>` HTTP header on every call to our Open Data Hub APIs.

The token has a validity of 1 hour, so it's best to automate it's request in your application.

These credentials will only be valid during the Bootcamp event

>[!IMPORTANT]
>If you don't pass a token, you are limited to 5 days of time series history per call, or 100 days when passing a `Referer` http header.  
>
>Request beyond that limit will result in a HTTP 429 Too Many Requests
See [quota limits](https://github.com/noi-techpark/opendatahub-docs/wiki/Historical-Data-and-Request-Rate-Limits) and [Http Referer](https://github.com/noi-techpark/opendatahub-docs/wiki/Http-Referer)


### More information
You can find more information on the API format here:  
[Swagger Ninja API](https://mobility.api.opendatahub.com)  
[Open Data Hub documentation](https://opendatahub.readthedocs.io/en/latest/mobility-tech.html)  
[Time series API README](https://github.com/noi-techpark/opendatahub-timeseries-api/blob/main/README.md)  