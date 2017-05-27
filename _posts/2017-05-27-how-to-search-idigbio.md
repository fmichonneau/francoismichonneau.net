---
layout: single
title: "How to use the iDigBio portal to look for taxa?"
date: 2017-05-27
type: post
published: true
categories: ["Tutorial"]
tags: ["Demonstration", "iDigBio"]
excerpt: "An illustrated demonstration on how to get taxa from a specific geographic zone using the iDigBio portal."
---


iDigBio is an NSF-funded project that aims (among other things) at centralizing information about museum specimens. Here, I illustrate how to use the data portal to obtain the list of specimens found in the database for a given geographic area. For our example, we will search for all extent (i.e., non-fossil) echinoderm (sea stars, sea urchins, sea cucumbers, brittle stars and sea lilies) specimens collected in French Polynesia.

* Go to the [iDigBio portal](https://portal.idigbio.org) and click on "Advanced Search"

![idigbio advanced search]({{ base.path }}/images/2017-05-27-idigbio-advanced-search.png)

* Because we want all the echinoderms, we will search all records with the phylum "Echinodermata". To do this, under the "Filter" tab, click on "Add a field", and click on "Taxonomy > Phylum". This will add a search box for the phylum.

![idigbio add phylum to search]({{ base.path }}/images/2017-05-27-add-phylum.png)

* Start typing "echinodermata" in this new box, and click on the word as it autocompletes.

![idigbio type phylum]({{ base.path}}/images/2017-05-27-type-phylum.png)

* Because there are many echinoderm fossils in the iDigBio database, and we are only interested in extent specimens, we will restrict our search to exclude fossils. To do this, under the "Add a field" dropdown menu, click on "Specimen > Basis of Record".

![idigbio basis of record]({{ base.path }}/images/2017-05-27-basis-record.png)

* In this next box, start typing "preservedspecimen" and click on the word as it autocompletes.

* To restrict our search to only include French Polynesia, click on the square icon in the top-left corner of the map, and draw a rectangle that encompass the geographic area you are intersted in.

![idigbio map]({{ base.path }}/images/2017-05-27-idigbio-map.png)

![idigbio draw search area]({{ base.path}}/images/2017-05-27-draw-map.png)

* The list under the map now includes all the specimens in iDigBio that match all your search criteria: extent Echinoderm collected in this particular geographic area.

![idigbio results]({{ base.path }}/images/2017-05-27-idigbio-results.png)


* You can download all the data associated with these specimens by clicking on the "Download" tab and adding your email address. Depending on the size of your query it will take between a few seconds to a few minutes to download the data or get a link to download it in your inbox.

![idigbio download]({{ base.path }}/images/2017-05-27-idigbio-download.png)

* **BONUS** if you click under the "Media" tab, with a little bit of luck, you might be able to see pictures of the specimens:

![idigbio media]({{ base.path }}/images/2017-05-27-idigbio-media.png)

## A few additional remarks

* Country data can be messy, and places like French Polynesia are rarely entered in the same way in databases across institutions. Therefore, searching using the words "French Polynesia" in one of the locality field will rarely give you all the results.

* On the other hand, searching using the map like I demonstrate here will not give you all the results either, because only the records that have geographic information (latitude and longitude) will be included.

* Double check your results carefully.
