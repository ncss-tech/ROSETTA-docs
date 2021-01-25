# ROSETTA-docs

Dylan Beaudette, Richard Reid, Todd Skaggs

2021-01-12

Introduction

This document outlines how to query soils data and access a new, online, interfae to the ROSETTA model for predicting hydraulic parameters from soil properties. The ROSETTA API was developed by Dr. Todd Skaggs (USDA-ARS) and links to the work of Zhang and Schaap, (2017). 

The ROSETTA model relies on a minimum of 3 soil properties, with increasing (expected) accuracy as additional properties are included:

* required, sand, silt, clay: USDA soil texture separates (percentages) that sum to 100%
* optional, bulk density (any moisture basis): mass per volume after accounting for >2mm fragments, units of gm/cm3
* optional, volumetric water content at 33 kPa: roughly “field capacity” for most soils, units of cm3/cm3
* optional, volumetric water content at 1500 kPa: roughly “permanent wilting point” for most plants, units of cm3/cm3

Soil properties must be described, in order, via vars argument. The API does not use the names but column ordering must follow: sand, silt, clay, bulk density, volumetric water content at 33kPa (1/3 bar), and volumetric water content at 1500kPa (15 bar).

For engineering applications, current processes being used to access authoritative soils data, predict hydraulic parameters, and subsequently calculate lateral effects distances are time consuming and a considerable amount of pre and post processing are required. ROSETTA hydraulic outputs included in the web service and API include output parameters of importance for engineering applications.

Through the ROSETTA REST API, a “proof of concept” python script can be used in IDLE to automate things and avoid the manual web interface. IDLE is python’s integrated development environment and comes with ArcPro however, python is a universal language and can be used in many different applications.

(ROSETTA 1) ROSETTA: a computer program for estimating soil hydraulic parameters with hierarchical pedotransfer functions - Schaap, et al.
(ROSETTA 2) Comparison of Models for Indirect Estimation of Water Retention and Available Water in Surface Soils - Schaap, et al. (this one is called version 1.3 in the software the engineers currently have)
(ROSETTA3) Weighted recalibration of the Rosetta pedotransfer model with improved estimates of hydraulic parameter distributions and summary statistics (Rosetta3) - Zhang, Schaap

ROSETTA Version 1 is the original. Version 3 is the newest.  Version 2 is currently what the engineers use to estimate the hydraulic parameters used in LE equations. This version has been added which is a bit different from version 1 and 3. Additionally, the new API allows logarithmic or linear output parameters to be returned. 

Looking to the future, I’m exploring ways to automate the other puzzle piece to lateral effects distance determination which is the soil Hydrogeomorphic wetland classification into a web service that could be “called” along with the authoritative soil data.
