# ROSETTA-docs

Dylan Beaudette, Richard Reid, Todd Skaggs

2021-01-12

## Introduction

This document outlines how to query soils data and access a new, online, interface to the ROSETTA model for predicting hydraulic parameters from soil properties. The ROSETTA API and web service were developed by Dr. Todd Skaggs (USDA-ARS) and links to the work of Zhang and Schaap, (2017). ROSETTA can be used to estimate the Van Genuchten hydraulic parameters that include the following:

theta_r - the residual soil water content, (cm3 cm-3)

theta_s - the saturated soil water content, (cm3 cm-3)

alpha_(1/cm) - parameter of the van Genuchten equation corresponding approximately to the inverse of the air-entry value, (cm-1)

n - the empirical shape-defining parameters in the van Genuchten equation, (dimensionless) 

Ksat_(cm/d) the effective saturated hydraulic conductivity, (cm day-1);


The ROSETTA model relies on a minimum of 3 soil properties, with increasing (expected) accuracy as additional properties are included:

* required, sand, silt, clay: USDA soil texture separates (percentages) that sum to 100%
* optional, bulk density (any moisture basis): mass per volume after accounting for >2mm fragments, units of gm/cm3
* optional, volumetric water content at 33 kPa: roughly “field capacity” for most soils, units of cm3/cm3
* optional, volumetric water content at 1500 kPa: roughly “permanent wilting point” for most plants, units of cm3/cm3

Soil properties must be described, in order, via vars argument. The API does not use logical names but column ordering must follow: sand, silt, clay, bulk density, volumetric water content at 33kPa (1/3 bar), and volumetric water content at 1500kPa (15 bar).

## Background and Practical Application

For engineering applications, current processes being used to access authoritative soils data, predict hydraulic parameters, and subsequently calculate lateral effects distances are time consuming and a considerable amount of pre and post processing are required. ROSETTA hydraulic outputs included in the web service and API include output parameters of importance for engineering applications.

A simplified version of ROSETTA is available as a web service and accepts user input soils parameters that can be manually inserted or pasted from an external table of soil parameters. The web service can be found [here](https://www.handbook60.org/rosetta/). An example of the ROSETTA web service interface is shown in Figure 1 below.

* Figure 1 - Web Service Interface of ROSETTA example
![Example: ROSETTA Web Service](https://github.com/ncss-tech/ROSETTA-docs/blob/main/static-figures/Rosetta_web%20service.png)

Through the ROSETTA REST API, a “proof of concept” python script can be used in IDLE to automate things and avoid the manual web interface. IDLE is python’s integrated development environment and comes with ArcPro however, python is a universal language and can be used in many different applications.

## Versioning

(ROSETTA 1) ROSETTA: a computer program for estimating soil hydraulic parameters with hierarchical pedotransfer functions - Schaap, et al(https://www.ars.usda.gov/arsuserfiles/20360500/pdf_pubs/P1765.pdf)

(ROSETTA 2) Comparison of Models for Indirect Estimation of Water Retention and Available Water in Surface Soils - Schaap, et al. (this one is called version 1.3 in the software the engineers currently have)

(ROSETTA3) Weighted recalibration of the Rosetta pedotransfer model with improved estimates of hydraulic parameter distributions and summary statistics (Rosetta3) - Zhang, Schaap

ROSETTA Version 1 is the original. Version 3 is the newest.  Version 2 is currently what the engineers use to estimate the hydraulic parameters used in LE equations. This version has been added which is a bit different from version 1 and 3. Additionally, the new API allows logarithmic or linear output parameters to be returned. 

## Testing and Validation 

For testing and validation purposes a "test" dataset must be created and used to predict hydraulic output parameters for each version of the ROSETTA model.

The following query can be used in [Soil Data Access](https://sdmdataaccess.nrcs.usda.gov/Query.aspx) to assemble a test dataset of the soil parameters used in the ROSETTA model:

"SELECT areasymbol, areaname, musym, muname, mapunit.mukey, component.cokey, compname, comppct_r, majcompflag, hydricrating, chorizon.hzname, chorizon.hzdept_r,  chorizon.hzdepb_r, chorizon.om_r, chorizon.ksat_r AS Ksat_um_per_sec, chorizon.sandtotal_r, chorizon.silttotal_r, chorizon.claytotal_r, chorizon.dbthirdbar_r, chorizon.wthirdbar_r / 100   AS wthirdbar_decimal, chorizon.wfifteenbar_r / 100 AS wfifteenbar_decimal

FROM legend

INNER JOIN mapunit ON legend.lkey=mapunit.lkey AND LEFT (areasymbol, 2) IN  ('MN', 'SD', 'ND', 'IA')
INNER JOIN component ON mapunit.mukey=component.mukey AND hydricrating = 'Yes' AND majcompflag = 'Yes'
LEFT OUTER JOIN chorizon ON component.cokey=chorizon.cokey"

## Future Projects
Looking to the future, I’m exploring ways to automate the other puzzle piece to lateral effects distance determination which is the soil Hydrogeomorphic wetland classification into a web service that could be “called” along with the authoritative soil data.
