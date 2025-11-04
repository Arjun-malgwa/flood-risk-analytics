# Data Sources & Acquisition

## Overview

This project integrated data from **5 primary sources** to create a comprehensive flood risk dataset for Montpelier, VT.

---

## 1. Montpelier Address Fabric

**Source**: City of Montpelier, Vermont - Property Records  
**Access Date**: September 2024  
**Records**: 10,311 properties  
**Format**: CSV  
**Cost**: Free (public records request)

### Fields Obtained:
- Unique Property ID
- Street Address
- City, State, ZIP Code
- Property Type (Residential, Commercial, Mixed-Use, Municipal)
- Parcel ID (GIS identifier)

### Acquisition Method:
1. Submitted FOIA request to Montpelier City Clerk's office
2. Received bulk export of property assessment database
3. Filtered to active properties (excluded demolished/pending)

### Data Quality:
- ✅ Complete address coverage
- ✅ No missing property IDs
- ⚠️ 127 properties had non-standard addresses (manually corrected)
- ⚠️ Coordinates not included (required geocoding)

### Use in Project:
Served as base layer for all geospatial analysis. Each property was geocoded to obtain latitude/longitude coordinates.

---

## 2. Precisely Geocoding API

**Source**: [Precisely GeoAddressing API](https://docs.precisely.com/docs/sftw/hadoop/landingpage/APIs/GeocodeAPI/GeocodeAPI.html)  
**Access Date**: September - October 2024  
**API Version**: v1  
**Cost**: Educational license (free for academic use)

### Service Used:
**Geocode Service (POST)** - Standardizes addresses and returns coordinates

### API Endpoint:
```
POST https://api.precisely.com/geocode/v1/geocode
```

### Request Parameters:
```json
{
  "preferences": {
    "returnAllCandidateInfo": false,
    "fallbackToGeographic": true,
    "maxReturnedCandidates": 1
  },
  "addresses": [
    {
      "mainAddressLine": "123 State St",
      "city": "Montpelier",
      "stateProvince": "VT",
      "postalCode": "05602"
    }
  ]
}
```

### Response Fields:
- `latitude`: Decimal degrees (NAD83)
- `longitude`: Decimal degrees (NAD83)
- `precisionLevel`: Rooftop, Parcel, Street, ZIP
- `matchCode`: Confidence score

### Geocoding Results:
| Precision Level | Count | Percentage |
|-----------------|-------|------------|
| Rooftop         | 9,847 | 95.5%      |
| Parcel          | 312   | 3.0%       |
| Street          | 141   | 1.4%       |
| ZIP             | 11    | 0.1%       |

### Data Quality Issues:
- 141 properties geocoded to street centroid (insufficient address detail)
- 11 properties geocoded to ZIP centroid (missing house numbers)
- Manual verification performed for all non-rooftop matches

### Use in Project:
Converted text addresses to precise geographic coordinates, enabling spatial analysis and API calls to Precisely FloodRisk.

---

## 3. Precisely FloodRisk API (GraphQL)

**Source**: [Precisely Property Intelligence - FloodRisk](https://docs.precisely.com/docs/sftw/hadoop/landingpage/APIs/PropertyIntelligence/PropertyIntelligence.html)  
**Access Date**: October 2024  
**API Version**: GraphQL  
**Cost**: Educational license (250,000 requests/month limit)

### Service Used:
**Property Attributes by Address** - Enriches property data with flood risk metrics

### GraphQL Query:
```graphql
query PropertyFloodRisk {
  propertyAttributesByAddress(
    address: {
      mainAddressLine: "123 State St"
      city: "Montpelier"
      stateProvince: "VT"
    }
    preferences: {
      propertyAttributes: [FLOOD_RISK, BUILDING_CHARACTERISTICS, DISTANCE_FACTORS]
    }
  ) {
    propertyAttributes {
      floodRiskData {
        floodZone
        nearestWaterbody
        Year100FloodZoneDistanceFeet
        DistanceToNearestWaterbodyFeet
        AddressLocationElevationFeet
      }
      buildingData {
        buildingConstructionType
        exteriorWalls
        roofCover
        livingSquareFootage
      }
      valuation {
        totalMarketValue
      }
    }
  }
}
```

### Key Metrics Obtained:

#### Flood Risk Metrics:
- **FloodZone**: FEMA designation (AE, A, X, SHX)
  - AE = 1% annual chance, base flood elevation determined
  - A = 1% annual chance, no base flood elevation
  - X = 0.2% annual chance (minimal risk)
  - SHX = Moderate risk, shaded on FIRM maps

- **Year100FloodZoneDistanceFeet**: Distance to nearest 100-year flood zone boundary
  - Mean: 2,847 feet
  - Min: 0 feet (375 properties in AE zone)
  - Max: 8,934 feet

- **DistanceToNearestWaterbodyFeet**: Straight-line distance to water
  - Primary waterbody: Winooski River
  - Mean: 1,456 feet
  - Properties < 500ft: 847 (8.2%)

- **AddressLocationElevationFeet**: Above sea level (NAVD88 datum)
  - Mean: 412 feet
  - Min: 324 feet (riverside properties)
  - Max: 876 feet (hillside properties)

#### Building Characteristics:
- Construction Type: Frame (72%), Masonry (18%), Concrete (10%)
- Exterior Walls: Wood Siding (63%), Vinyl (22%), Brick (11%), Other (4%)
- Roof Cover: Asphalt Shingle (81%), Metal (14%), Other (5%)

### API Call Statistics:
- **Total API calls**: 10,311 (one per property)
- **Successful responses**: 10,170 (98.6%)
- **Failed/timeout**: 141 (1.4%) - properties outside API coverage area
- **Rate limiting**: Batched requests at 100/minute to avoid throttling

### Data Quality:
- ✅ High-quality flood zone data (sourced from FEMA FIRM maps)
- ✅ Elevation data matches USGS 3DEP within ±5 feet
- ⚠️ 141 properties returned incomplete data (rural areas outside API coverage)
- ⚠️ Historical flood events not included (only current risk metrics)

### Use in Project:
Primary source of flood risk features for clustering model. Distance and elevation metrics were most predictive variables.

---

## 4. FEMA National Flood Insurance Program (NFIP) - Vermont PIF

**Source**: [FEMA OpenFEMA Data](https://www.fema.gov/openfema-data-page/fima-nfip-redacted-policies-v2)  
**Access Date**: November 2024  
**Dataset**: Policies in Force (PIF) - Vermont  
**Format**: CSV  
**Cost**: Free (public open data)

### Dataset Details:
- **Total Vermont policies**: 555 active policies (as of Sept 2024)
- **Time period**: Active policies as of query date
- **Privacy**: Addresses redacted to Census block level (no PII)

### Fields Used:
| Field | Description | Example |
|-------|-------------|---------|
| `policyEffectiveDate` | Policy start date | 2023-06-15 |
| `totalBuildingInsuranceCoverage` | Policy value (USD) | $250,000 |
| `propertyValue` | Assessed value (USD) | $350,000 |
| `deductibleAmountBuildingCoverage` | Deductible (USD) | $5,000 |
| `elevatedBuildingIndicator` | First floor elevated? | Y/N |
| `primaryResidence` | Is primary home? | Y/N |
| `constructionType` | Frame/Masonry/Concrete | Frame |
| `numberOfFloorsInTheInsuredBuilding` | Story count | 2 |

### Filtering & Processing:
1. **Original dataset**: 555 Vermont policies
2. **Filtered to properties with complete data**: 216 properties
   - Removed: Missing property values (214)
   - Removed: Missing living area (89)
   - Removed: Duplicates (36)
3. **Final dataset for underinsurance model**: 216 properties

### Calculated Metrics:
```python
# Insurance-to-Value (ITV) Ratio
ITV = totalBuildingInsuranceCoverage / propertyValue

# Underinsurance Flag
underinsured = 1 if ITV < 0.8 else 0

# Underinsurance Gap (for underinsured properties)
gap = propertyValue * 0.8 - totalBuildingInsuranceCoverage
```

### Distribution:
| ITV Range | Count | Percentage |
|-----------|-------|------------|
| < 0.5 (Severely underinsured) | 37 | 17.1% |
| 0.5 - 0.8 (Underinsured) | 47 | 21.8% |
| 0.8 - 1.0 (Adequate) | 89 | 41.2% |
| > 1.0 (Overinsured) | 43 | 19.9% |

**Key Finding**: 38.9% of Vermont NFIP policies are underinsured (ITV < 0.8)

### Limitations:
- ⚠️ Small sample size (216 properties with complete data)
- ⚠️ Addresses redacted (could not directly link to Montpelier properties)
- ⚠️ Only includes NFIP policies (excludes private flood insurance)
- ✅ Representative of Vermont flood insurance landscape

### Use in Project:
Training data for Random Forest underinsurance prediction model. Model learned patterns of underinsurance based on property characteristics.

---

## 5. FEMA Flood Insurance Rate Maps (FIRM)

**Source**: [FEMA Map Service Center](https://msc.fema.gov/)  
**Map Panel**: 50023C0222H (Montpelier, VT)  
**Effective Date**: December 2, 2015  
**Format**: GeoJSON (digitized flood zones)  
**Cost**: Free (public)

### Flood Zones in Montpelier:

| Zone | Description | Properties | Risk Level |
|------|-------------|------------|------------|
| **AE** | 1% annual chance, base flood elevation shown | 375 | High |
| **SHX** | 0.2% annual chance (moderate risk, shaded) | 168 | Moderate-High |
| **A** | 1% annual chance, no base elevation | 43 | High |
| **X** | 0.2% annual chance or less | 9,584 | Low |

### Map Metadata:
- **Projection**: NAD83 / Vermont State Plane (EPSG:32145)
- **Vertical Datum**: NAVD88
- **Contour Interval**: 2 feet
- **Hydrology**: Based on 2011 stream gauge data

### Historical Flood Events (from FIRM documentation):
- **July 1927**: Major flood, pre-dam construction
- **November 1927**: Record flood (pre-Waterbury Dam)
- **March 1936**: Spring melt flood
- **April 2011**: Tropical Storm Irene precursor
- **August 2011**: Tropical Storm Irene (1% flood exceeded)
- **July 2023**: Flash flood (downtown inundation)
- **July 2024**: Anniversary flood (Barre Street, Berlin)

### Data Processing:
1. Downloaded FIRM panel as PDF
2. Digitized flood zones using QGIS
3. Exported as GeoJSON (EPSG:4326 for compatibility)
4. Spatial join with geocoded properties

### Spatial Analysis:
```python
# Identify properties in flood zones
gdf_properties.sjoin(gdf_flood_zones, how='left', predicate='within')

# Calculate distance to flood zone boundary
gdf_properties['flood_zone_dist'] = gdf_properties.geometry.distance(
    gdf_flood_zones[gdf_flood_zones['zone'].isin(['AE', 'A'])].unary_union
)
```

### Validation:
- ✅ Cross-referenced with Precisely FloodRisk API (99.2% agreement on zone designation)
- ✅ Manually verified 50 random properties using FEMA Map Service Center
- ✅ Aligned with Vermont ANR flood hazard data

### Use in Project:
Ground truth for flood zone classification. Used to validate Precisely API responses and as independent variable in clustering.

---

## Data Integration Pipeline

### Step 1: Address Standardization
```
Montpelier Address Fabric (10,311)
    ↓
Precisely Geocoding API
    ↓
Geocoded Properties (10,311 with lat/lon)
```

### Step 2: Flood Risk Enrichment
```
Geocoded Properties (10,311)
    ↓
Precisely FloodRisk API (GraphQL)
    ↓
Enriched Properties (10,170 - 1.4% failed)
```

### Step 3: FEMA Validation
```
Enriched Properties (10,170)
    ↓
Spatial Join with FEMA FIRM (GeoJSON)
    ↓
Validated Flood Zones (10,170)
```

### Step 4: Clustering
```
Validated Flood Zones (10,170)
    ↓
Feature Selection (PCA) → 8 key features
    ↓
K-means Clustering (k=5)
    ↓
Clustered Properties (10,170 with risk tiers)
```

### Step 5: Underinsurance Modeling
```
Vermont NFIP PIF (555)
    ↓
Data Cleaning & ITV Calculation
    ↓
Complete Cases (216)
    ↓
Random Forest Classifier
    ↓
Underinsurance Predictions
```

---

## Data Lineage & Reproducibility

### Version Control:
- Raw data stored in private S3 bucket (not in Git)
- Data processing scripts tracked in Git
- Output checksums recorded in `data_manifest.json`

### Reproducibility:
To reproduce this analysis:
1. Obtain Precisely API credentials (educational license available)
2. Request Montpelier property data from city clerk
3. Download FEMA FIRM maps and Vermont PIF data (public)
4. Run `src/data_pipeline.py` with your API keys in `.env`

**Estimated time**: 4-6 hours (mostly API calls)  
**Estimated cost**: $0 (with educational licenses)

---

## Data Governance & Ethics

### Privacy Protections:
- ✅ No personally identifiable information (PII) published
- ✅ Addresses aggregated to Census block level in public reports
- ✅ Individual policy details not disclosed
- ✅ Synthetic sample data provided for code demonstration

### Data Use Agreements:
- Precisely APIs: Academic use only, no redistribution
- FEMA data: Public domain, no restrictions
- Montpelier property data: Research use approved by city clerk

### Citation Requirements:
When using aggregated statistics from this project, cite as:
```
Malgwa, A., et al. (2024). Flood Risk Analytics for Montpelier, VT. 
Drexel University  BSAN 710 Capstone Project.
```

---

## Contact

For questions about data sources or access:

**Arjun Malgwa**  
📧 arjun.malgwa@gmail.com  
🔗 [linkedin.com/in/arjunmalgwa](https://linkedin.com/in/arjunmalgwa)
