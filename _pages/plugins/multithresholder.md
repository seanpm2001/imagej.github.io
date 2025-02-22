---
mediawiki: MultiThresholder
name: "MultiThresholder"
title: MultiThresholder
categories: [Segmentation]
release-date: "25 January 2007"
initial-release-date: "21 July 2005"
website: "https://imagej.net/ij/plugins/multi-thresholder.html"
dev-status: "unknown"
team-founder: 'Kevin (Gali) Baler'
---

{%- capture removed-from-fiji -%}
This plugin was removed, as the [Auto Threshold](/plugins/auto-threshold)
plugin from {% include person id='landinig' %} does a better job.
{%- endcapture removed-from-fiji -%}
{% include notice icon="info" content=removed-from-fiji %}

{% include info-box filename=' [Multi_Thresholder.jar](https://imagej.net/ij/plugins/download/jars/Multi_Thresholder.jar) (30,729 Bytes)' source='in .jar file'  %}

## Purpose

This plugin allows the user to apply four different automatic thresholding
algorithms. The four algorithms are ImageJ's built in IsoData algorithm,
[Maximum Entropy Threshold](/plugins/maximum-entropy-threshold),
[Otsu Thresholding](Otsu_Thresholding), and
[Mixture Modeling](/plugins/mixture-modeling-thresholding). More information
about thresholding and the algorithms employed here can be found at the
[Image Thresholding Tutorial](http://www.ph.tn.tudelft.nl/Courses/FIP/noframes/fip-Segmenta.html).

## Usage

This plugin is quite convenient, for it groups different techniques in one. It
has nevertheless a major drawback when used on stacks. When operating on a
stack, it computes the threshold on the *single* image currently displayed, and
will apply this threshold value to *all* of the other images, which might not
be relevant.
