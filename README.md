# DSS_Germany_Facility_Location

## Data Cleaning & Preparation (Step-by-step)

This section explains exactly how we transformed the raw datasets into the cleaned demand dataset and the all-to-all distance file that will be used in downstream analysis/optimization.

### 0) Input (Raw Data)

We used two raw sources:

data/raw/germany_cities_stadtrecht_2023.csv
Contains German cities with administrative state, population statistics and area/density columns.
Key columns used: state, city, population_total

data/raw/Deutschland_Cities.csv
Contains city coordinates and metadata.
Key columns used: city, lat, lng

### 1) Load & Basic Quality Checks

We loaded both datasets and performed quick sanity checks:

inspected shape and column types

checked missing values (isna().sum())

checked duplicates (duplicated().sum())

Result highlights:

Population dataset had no missing values for population.

There were no duplicates.

city_designation had many missing values, so it was not used.

### 2) Column Selection & Standardization

From the population dataset, we selected only the columns needed for modeling:

state

city

population_total → renamed to population

Optional columns like area_km2 and per_km2 were kept only if needed later.

We standardized column naming to keep the dataset consistent and model-agnostic.

### 3) City Name Normalization (Critical for Merging)

A direct merge on city was not reliable because city names can differ in formatting across sources (e.g., umlauts, parentheses, abbreviations, alternative names).

We created a normalized join key to ensure stable matching.

Normalization rules applied (conceptually):

lowercasing and trimming whitespace

German character normalization: ä→a, ö→o, ü→u, ß→ss

removing parentheses content (e.g., Halle (Saale) → Halle)

taking the first part before / for alternative names (e.g., Cottbus/Chóśebuz → Cottbus)

removing punctuation and collapsing multiple spaces

Output: a new key column:

city_key2 (normalized city identifier)

### 4) Merge Population + Coordinates

We merged the population dataset with the coordinates dataset using the normalized key:

From coordinates: lat, lng → renamed to latitude, longitude

The merged output provides:

city & state labels

population

latitude/longitude for distance computations

We dropped any rows missing coordinates and removed duplicates by city_key2.

### 5) City Filtering (Why we have 368 cities)

After merging and ensuring valid coordinates, we ended up with 368 cities in the cleaned dataset.

This is the final set used for:

demand weights

distance computation

(If you want to adjust coverage later, change the filtering strategy—e.g., population threshold or include more coordinate matches.)

## 6) Demand Weight Creation

We created a normalized demand weight from population:

demand_weight_i = population_i / sum(population_all)

Why this is done:

ensures total demand sums to 1

allows direct use in demand-weighted objectives (p-median / optimization)

makes the dataset scale-independent and comparable across scenarios

Sanity check:

sum(demand_weight) ≈ 1.0

### 7) Cleaned Dataset Output

Final cleaned dataset saved as:

data/cleaned/clean_cities_with_population_coords.csv

Columns:

city_key2 (normalized join key)

city

state

population

demand_weight

latitude

longitude

This file is the primary input for any optimization/ML work.

### 8) All-to-All Distance Computation

We computed geographic distances between every pair of cities using the Haversine formula (great-circle distance):

Output unit: kilometers

Earth radius assumed: 6371 km

To reduce file size and redundancy, we stored distances in an upper-triangle format:

no self-distances (city to itself)

no duplicates (A–B included once; B–A not repeated)

This format is sufficient because distances are symmetric:

dist(A,B) = dist(B,A)

### 9) Distance Output

All-to-all distances saved as:

data/cleaned/distance_all_pairs_uppertriangle.csv

Columns:

city_a

city_b

distance_km

### 10) How to Use These Outputs

Use clean_cities_with_population_coords.csv as the master demand table.

Use distance_all_pairs_uppertriangle.csv to:

build a distance matrix (NxN) if needed

compute nearest facilities / clusters

feed optimization models (p-median) after selecting candidate facilities

run any network-based or ML feature engineering that relies on distances


