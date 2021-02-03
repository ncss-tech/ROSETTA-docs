# ROSETTA-docs

![ROSETTA Banner](https://github.com/ncss-tech/ROSETTA-docs/blob/main/static-figures/Rosetta_Banner.png)

Richard Reid, Dylan Beaudette, Todd Skaggs

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

ROSETTA hydraulic parameters are also used in a variety of other ways including:
* HYDRUS-3D Simulation of Soil Water Dynamics in Drip-Irrigated Settings
* Land Surface Modeling
* Agroecosystem models
* regional and global climate models, and 
* Numerical weather prediction models 

A simplified version of ROSETTA is available as a web service and accepts user input soils parameters that can be manually inserted or pasted from an external table of soil parameters. The web service can be found [here](https://www.handbook60.org/rosetta/). An example of the ROSETTA web service interface is shown in Figure 1 below.

* Figure 1 - Web Service Interface of ROSETTA example
![Example: ROSETTA Web Service](https://github.com/ncss-tech/ROSETTA-docs/blob/main/static-figures/Rosetta_web%20service.png)

Through the ROSETTA REST API, a “proof of concept” python script can be used in IDLE to automate things and avoid the manual web interface. IDLE is python’s integrated development environment and comes with ArcPro however, python is a universal language and can be used in many different applications.

There are many options for using the ROSETTA REST API. Two methods will be explored here. One method is to query the authoritative soils data and run the ROSETTA predictions simultaneously. Another method is to place an existing soils input table into the script and run the ROSETTA predictions. There are pros and cons of using each method. 

### Option 1 - Query the authoritative soils data and run the ROSETTA predictions simultaneously in Python

"""
Proof-of-concept for generating soil hydraulic parameters using
SDMDataAccess and the Rosetta web api.

"""
import requests
import sys
from typing import List


def rosetta_url(rosetta_version: int) -> str:
    return f"http://www.handbook60.org/api/v1/rosetta/{rosetta_version}"


def query(areasymbol: str) -> str:
    return f"""
		SELECT
                           areasymbol,
                           muname,
		           musym,
		           mapunit.mukey,
                           component.cokey
			   compname,
			   comppct_r,
			   chorizon.hzname,
			   chorizon.hzdept_r,
			   chorizon.hzdepb_r,
			   chorizon.sandtotal_r,
			   chorizon.silttotal_r,
			   chorizon.claytotal_r,
			   chorizon.dbthirdbar_r,
			   chorizon.wthirdbar_r / 100   AS wthirdbar_decimal,
			   chorizon.wfifteenbar_r / 100 AS wfifteenbar_decimal
		FROM    (legend
		INNER JOIN (mapunit
		INNER JOIN component
		ON mapunit.mukey = component.mukey)
		ON legend.lkey = mapunit.lkey)
		INNER JOIN chorizon
		ON component.cokey = chorizon.cokey
		WHERE  comppct_r >= 10 AND legend.areasymbol LIKE '{areasymbol}'
    """


def request_ssurgo_tabular(areasymbol: str) -> List[List]:

    url = "https://SDMDataAccess.sc.egov.usda.gov/Tabular/post.rest"
    params = {"format": "JSON+COLUMNNAME", "query": query(areasymbol)}
    r = requests.post(url, data=params)

    if not r.ok:
        print(f"Error!\nStatus code: {r.status_code}\nMessage:\n{r.text}")
        sys.exit()

    return r.json()["Table"]


def main():
    rosetta_version = 2
    area_symbol = "SD%"

    print(f"Requesting SSURGO data for {area_symbol} ...")
    ssurgo_data = request_ssurgo_tabular(area_symbol)
    print("Success!")

    # this next line assumes the fields sa, si, cl, bd, th33, th1500 were at
    # the end of the ssurgo SELECT (and in that order)
    rosetta_input = [row[-6:] for row in ssurgo_data[1:]]

    print("Requesting Rosetta estimates ...")
    r = requests.post(rosetta_url(rosetta_version), json={"X": rosetta_input})

    if not r.ok:
        print(f"Error!\nStatus code: {r.status_code}\nMessage:\n{r.text}")
        sys.exit()
    print("Success!")

    # We are done. Now it is just a question of what to do with the results.
    # Here I dump everything to csv

    rosetta_header = "thr ths log10_alpha_(1/cm) log10_npar log_Ksat_(cm/d)".split()
    import csv

    with open("SDA_ROSETTA_results.csv", "w", newline="") as csvfile:
        writer = csv.writer(csvfile)
        writer.writerow(ssurgo_data[0] + rosetta_header + ["model_code"])
        for a, b, c in zip(
            ssurgo_data[1:], r.json()["van_genuchten_params"], r.json()["model_codes"]
        ):
            writer.writerow(a + b + [c])


if __name__ == "__main__":
    main()



### Option 2 - Use an existing soils input data table to run ROSETTA predictions in Python

from typing import List, Union                                                           
import requests                                                                          
                                                                                         
DATA = {                                                                                 
    "X": [                                                                               
        [49, 7.0, 44, 0.99, 0.29011, 0.0261],                                            
        [42, 30, 28, 1.78, 0.2434, 0.09031],                                             
        [30, 46, 24, 1.02],                                                              
        ["30", "46", "24", "1.02"],                                                      
        [34, 38, 28, 1.23, 0.21066, "BAD"],                                              
        [22, 43, 35, 1.56, 0.16495, 0.02243],                                            
        [85, 9.0, 6.0, 1.08, -99, 0.07483],                                              
        [31, 44, 25, 1.44, 0.1977, 0.09442],                                             
        # next entry lacks  minimal required sa, si, cl data                             
        [12, 21, "NAN", 1.02, 0.18889, 0.00821],                                         
        [48, 11, 41, 1.80, 0.2677, 0.12446],                                             
        [82, 2.2, 16, 1.58, 0.21149, 0.12008],                                           
        [None, None, None, None, None, None],                                            
        [84, 16, 0.0, 1.08, 0.17206, 0.023],                                             
        [71, 24, 5.0, 1.88, 0.28051, 0.12799],                                           
        # next entry sa, si, cl are out of bounds, sum >> 100                            
        [142, 33, 25, 0.73, 0.22333, 0.12299],                                           
        [39, 22, 39, 0.68, 0.21911, 0.00119],                                            
    ]                                                                                    
}                                                                                        
                                                                                         
                                                                                         
def url(version: int) -> str:                                                            
    return f"http://www.handbook60.org/api/v1/rosetta/{version}"                         
                                                                                         
def to_nan(hydparams: List[Union[float, None]]) -> List[float]:                          
    """covert any None parameter values to nan"""                                        
    return [v if v is not None else float("nan") for v in hydparams]                     
                                                                                         
def to_linear(hydparams: List[float]) -> List[float]:                                    
    """convert alp, npar, ksat to linear values."""                                      
    hydparams[2:] = [10 ** v for v in hydparams[2:]]                                     
    return hydparams                                                                     
                                                                                         
                                                                                         
def pretty_print(parameters: List[List[float]], codes: List[int]) -> str:                   
    out = ("{:>10}" * 6).format(*"code thr ths alpha npar ksat".split())                 
    out += "\n"                                                                          
    fmt = "{:10}" + "{:10.5f}" * 5 + "\n"                                                
    for params, code in zip(parameters, codes):                                          
        out += fmt.format(code, *params)                                                 
    out += "\nUnits are alp = 1/cm, ksat = cm/d\n"                                       
    return out                                                                           
                                                                                         
                                                                                         
r = requests.post(url(version=2), json=DATA)

if r.ok:                                                                                 
    params = [to_nan(row) for row in r.json()["van_genuchten_params"]]                   
    params = [to_linear(row) for row in params]                                          
    print(pretty_print(params, r.json()["model_codes"]))                                 
else:                                                                                    
    print(r.status_code)                                                                 
    print(r.text)

## ROSETTA API Notes

### URL:

https://www.handbook60.org/api/v1/rosetta/{int:v}


where

v = 1, 2, or 3


e.g.:

https://www.handbook60.org/api/v1/rosetta/3

(Note: when making a request using the python Requests library, I had to use “http” rather than “https”. Not sure why.)

### JSON data

The POST request expects a json payload with the form

{‘X’: data}

where data is a list of lists,  [[,,,],[,,,],[,,,] … ]

Each sublist must be one of the following:
                             
Data                                                                                                             (Also known as)
 [sand%, silt%, clay%]                      (model code 2)                           
 [sand%, silt%, clay%, rho_b]               (model code 3)                                                         
 [sand%, silt%, clay%, rho_b, th33]         (model code 4)                                                        
 [sand%, silt%, clay%, rho_b, th33, th1500] (model code 5)                           

where:                                                                                   
                                                                                        
 sand%, silt%, and clay%       are the soil separates (sum to ~100)                           
 rho_b    is soil bulk density (gm/cm3)                                                    
   th33    is the soil volumetric water content at 33 kPa                                    
   th1500   is the  soil volumetric water content at 1500 kPa  

#### E.g., the json payload might be

{‘X’: [[30,40.5,29.5,1.6],[10,50,40],[90,5,5,1.7,0.22,0.09]]}

Note that even if there is only one sublist, ‘X’ still needs to be 2D list:

 {‘X’: [[30,40.5,29.5,1.6]]}

### Returned values

The returned json has the form

{“van_genuchten_params”: [[,,,,],[,,,,],...],
 “model_codes”: [,, …],
 “rosetta_version”: int
}

Each sublist in “van_genucthen_params”  is

[theta_r, theta_s, log10(alpha), log10(npar), log10(ksat)]

where

theta_r  : residual volumetric water content                                            
theta_s  : saturated volumetric water content                                           
alpha       : retention shape parameter (1/cm)                                                       
npar          : retention shape parameter                                                              
ksat        : saturated hydraulic conductivity (cm/day)

“model_codes” is the list of model codes used to predict the corresponding entry in “van_genuchten_params”

“rosetta_version” is the Rosetta version


### An example using curl:

curl -X POST -H "Content-Type:application/json" -d '{"X": [[50,40,10,1.6,0.25],[10,40,50,1.5,0.2]]}' "https://www.handbook60.org/api/v1/rosetta/3"

{"model_codes": [4, 3], "rosetta_version": 3, "van_genuchten_params": [[0.06930329923825358, 0.3554805457584735, -2.0910869111740067, 0.13999104372677112, 1.1086406024382627], [0.12952613311555153, 0.43802529604640894, -2.1426087306876362, 0.10828373362008699, 0.6976299524746705]]}




### An example using python requests:

import requests                                                                             
                                                                                         
DATA = {                                                                                 
    "X": [                                                                               
        [49, 7.0, 44, 0.99, 0.29011, 0.0261],                                            
        [42, 30, 28, 1.78, 0.2434, 0.09031],                                             
        [30, 46, 24, 1.02],                                                              
    ]                                                                                    
}                                                                                        
                                                                                         
                                                                                         
def url(version: int) -> str:                                                            
    return f"http://www.handbook60.org/api/v1/rosetta/{version}"                         
                                                                                         
r = requests.post(url(version=2), json=DATA)                                             
                                                                                         
if r.ok:                                                                                 
    print(r.json())                                                                      
else:                                                                                    
    print(r.status_code)                                                                    
    print(r.text)



### Results:

{'model_codes': [5, 5, 3], 'rosetta_version': 2, 'van_genuchten_params': [[0.03626029823435693, 0.5130283664640276, -2.0560562191308946, 0.14028088599077657, 1.9844066153092994], [0.04371623299088531, 0.3216468185313934, -2.160978983929374, 0.13509126645704977, 0.7097915645849571], [0.0800992063209251, 0.5159619107502098, -2.060414700720488, 0.13422539000745515, 1.957106692448703]]}


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
