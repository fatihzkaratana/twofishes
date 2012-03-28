A coarse, splitting geocoder in scala, with a legacy implementation in python.

What is a Geocoder?
===================

A geocoder is a piece of software that translates from strings to coordinates. "New York, NY" to "40.74,  -74.0". This is an implementation of a coarse (city level, meaning it can't understand street addresses) geocoder that also supports splitting (breaking off the non-geocoded part in the final response).

Overview
========

This geocoder was designed around the geonames data, which is relatively small, and easy to parse in a short amount of time in a single thread without much post-processing. Geonames is which is a collection of simple text files that represent political features across the world. The geonames data has the nice property that all the features are listed with stable identifiers for their parents, the bigger political features that contain them (rego park -> queens county -> new york state -> united states). In one pass, we can build a database where each entry is a feature with a list of names for indexing, names for display, and a list of parents.

I use mongo because it waas easy. It's nice the the data store reamins mutable during index building time, so we can supplement the geonames points with flickr bounding boxes. When serving from mongo, we can keep everything in memory and treat it like a glorified key-value store. If we were doing heavier processing on the incoming data, a mapreduce that spits out a static serving data file might make more sense. For now, this works.

When we parse a query, we do a rough recursive descent parse, starting from the left. If being used to split geographic queries like "pizza new york" we expect the "what" to be on the left. All of the features found in a parse must be parents of the smallest 

The geocoder currently may return multiple valid parses, however, it only returns the longest possible parses. For "Springfield, US" we will return multiple features that match that query (there are dozens of springfields in the US). It will not return a parse of "Springfield" near "US" with only US geocoded if it can find a longer parse, but it will return multiple valid interpretations of the longest parse.

The Data
========

Geonames is great, but not perfect. Southeast Asia doesn't have the most comprehensive coverage. It doesn't have bounding boxes, so we add those from http://code.flickr.com/blog/2011/01/08/flickr-shapefiles-public-dataset-2-0/ where possible. This file was generated by a now orphaned python script that lives at legacy-python/flickr/match_flickr.py. We attempted to geocode each feature name with the legacy python geocoder and then checked if the geonames point was within the bounds of the flickr feature.

Additionally, the mapbox folks generated a mapping from OSM features with geometry to geonameids, however OSM has very low polygon/bounding box coverage.

Geonames is licensed under CC-BY-SA http://www.geonames.org/
Flickr shapefiles are public domain 

Requirements
============
*   Scala
*   [Mongo](http://www.mongodb.org/display/DOCS/Quickstart)
*   curl

First time setup
================
*   git clone git@github.com:foursquare/twofish.git
*   cd twofish
*   ./init.sh
*   ./download-US.sh # or ./download-world.sh

Data import
===========
*   mongod --dbpath /local/directory/for/output/
*   ./init-database.sh # drops existing table and creates indexes
*   ./sbt "indexer/run-main com.foursquare.twofish.importers.geonames.GeonamesParser --parse_country US" # or ./sbt "indexer/run-main com.foursquare.twofish.importers.geonames.GeonamesParser --parse_world true"

Serving
=======
*   mongod --dbpath /local/directory/for/output/
*   ./sbt  "server/run-main com.foursquare.twofish.GeocodeFinagleServer --port 8080"
*   server should be responding to finagle-thrift on the port specified (8080 by default), and responding to http requests at the next port up: http://localhost:8081/?query=rego+park+ny http://localhost:8081/static/geocoder.html#rego+park

Future
======
I'd like to integrate more data from OSM and possibly an entire build solely from OSM. I'd also like to get supplemental data from the Foursquare database where possible. If I was feeling more US-centric, I'd parse the TIGER-line data for US polygons, but I'm expecting those to mostly be in OSM. 

Also US-centric are zillow neighborhood polygons, also CC-by-SA. I might add an "attribution" field to the response for certain datasources. I'm not looking forward to writing a conflater with precedence for overlapping features from different data sets.

Legacy Python Server
====================
I originally implemented this as a weekend python hack in a few hundred lines ... then added some features ... then some more ... and then it was way too much gross python with no object model. So I rewrote it entirely in scala. If you want to try running the python server, use importer.py in legacy-python/ and geocoder.py in legacy-python/

