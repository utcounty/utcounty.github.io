# Data Sources

## **Utah County Precinct Boundaries**:
Statewide ballot area polygons were downloaded from Utah’s Open Data Portal:

* **Dataset**: Utah VISTA Ballot Areas
* **URL**: https://opendata.gis.utah.gov/datasets/utah::utah-vista-ballot-areas/about
* **Original file**: VistaBallotAreas_-1176372527926423573.geojson

**Processing**:

* Load the statewide GeoJSON
* Filter to Utah County using `CountyID == 25`:
* Keep only required fields: `['CountyID', 'PrecinctID', 'SubPrecinctID', 'geometry']`
* Reproject for web mapping: `WGS84 (EPSG:4326) `
* Lightly simplify geometry for performance: `tolerance = 0.0001`

**Output**:
data/geo/utah_county_precincts.min.geojson

## Turnout Data

**2025-11-04 Utah County Municipal General Election**
* **Link**: https://electionresults.utah.gov/results/public/utah-county-ut/elections/general11042025/voters
* **File Name**: 20251104_utah_county_municipal_general_election.json
* **Original File Name**: export-general11042025.json


**2025-08-12 Utah County Municipal Primary Election**
* **Link**: https://electionresults.utah.gov/results/public/utah-county-ut/elections/primary08122025/voters
* **File Name**: 20250812_utah_county_municipal_primary_election.json
* **Original File Name**: export-primary08122025.json


**2024-11-05 Utah County General Election**
* **Link**: https://electionresults.utah.gov/results/public/utah-county-ut/elections/general11052024/voters
* **File Name**: 20241105_utah_county_general_election.json
* **Original File Name**: export-general11052024.json


**2024-06-25 Utah County Primary Election**
* **Link**: https://electionresults.utah.gov/results/public/utah-county-ut/elections/primary06252024/voters
* **File Name**: 20240625_utah_county_primary_election.json
* **Original File Name**: export-primary06252024.json


**2024-03-05 Utah County Presidential Preference Primary Election**
* **Link**: https://electionresults.utah.gov/results/public/utah-county-ut/elections/primary03052024/voters
* **File Name**: 20**Original File Name**: 240305_utah_county_presidential_preference_primary_election.json
* export-primary03052024.json

**2023-11-21 Utah County General Election**
* **Link**: https://electionresults.utah.gov/results/public/utah-county-ut/elections/2023-Nov-General/voters
* **File Name**: 20231121_utah_county_general_election.json
* **Original File Name**: export-2023-Nov-General.json