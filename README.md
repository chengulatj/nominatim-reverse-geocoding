# Nominatim Reverse Geocoding

## Description
This repository demonstrates how to use the **Nominatim API** from OpenStreetMap for reverse geocoding. It allows you to convert latitude and longitude coordinates into meaningful location details, specifically extracting the **county** in which each location resides.

The tool also handles coordinates in **Degrees, Minutes, Seconds (DMS)** format by converting them to decimal format for geocoding.

## Features
- Convert latitude and longitude from **Degree-Minutes-Seconds** to **decimal format**.
- Use the **Nominatim API** to reverse geocode the coordinates.
- Extract and store the **county** from the reverse geocoded data.
- Handle errors and timeouts during the API calls.

## Requirements
- Python 3.x
- Pandas (for data handling)
- Geopy (for Nominatim API)

## Installation

### 1. Install the Required Python Packages:
Run the following command to install the required Python packages:
```bash
pip install pandas geopy openpyxl
```

### Package Details:
'pandas': For data manipulation and analysis.
**geopy**: For geocoding using the Nominatim API.
**openpyxl**: For reading and writing Excel files.

## Code Explanation

Below is a step-by-step explanation of the code used in the colab notebook linked below:


### 1. Import Necessary Libraries

```python
import pandas as pd
import re
import time
from geopy.geocoders import Nominatim
from geopy.exc import GeocoderTimedOut, GeocoderUnavailable
```
- **pandas**: For data manipulation, specifically for reading and writing Excel files and handling the latitude/longitude coordinates.
- **re**: Used for regular expressions to parse the latitude and longitude in **Degrees, Minutes, Seconds (DMS)** format.
- **time**: Used to introduce delays when retrying API requests that time out.
- **geopy**: Provides the **Nominatim** geocoder, allowing reverse geocoding of latitude and longitude coordinates.
- **GeocoderTimedOut, GeocoderUnavailable**: Exceptions from **geopy** to handle timeout and service unavailability errors.

### 2. Read the Excel File

```python
locations = pd.read_excel('SL0202_Replacement_Locations.xlsx', skiprows=1)
```
- This line reads the input Excel file (`SL0202_Replacement_Locations.xlsx`) and loads it into a pandas DataFrame.  
- The `skiprows=1` argument skips the first row if it contains an extra header.


### 3. Extract Latitude and Longitude Columns
```python
latlongs = locations[['Latitude', 'Longitude']]
```
This subdataset creates a new DataFrame **latlongs** that contains only the Latitude and Longitude columns extracted from the original locations DataFrame.

### 4.  Define DMS to Decimal Conversion Function
```python
def dms_to_dd(dms_str):
    dms_str = str(dms_str)
    parts = re.split('[°\'"]+', dms_str)

    if len(parts) != 4:
        return float('nan')

    degrees = float(parts[0])
    minutes = float(parts[1])
    seconds = float(parts[2])
    direction = parts[3].strip()

    dd = degrees + minutes / 60 + seconds / 3600
    if direction in ['S', 'W']:
        dd *= -1
    return dd
```
- This function converts latitude and longitude coordinates from **DMS (Degrees, Minutes, Seconds)** format to **decimal** format.  
- The `re.split()` function is used to split the DMS string into degrees, minutes, seconds, and direction components.  
- The calculation converts the DMS components into a single decimal value.  
- The direction ('S' or 'W') makes the result negative for coordinates in the Southern or Western hemispheres.


### 5. Apply Conversion to Latitude and Longitude Columns

```python
latlongs.loc[:, 'Latitude_dec'] = latlongs.loc[:, 'Latitude'].apply(dms_to_dd)
latlongs.loc[:, 'Longitude_dec'] = latlongs.loc[:, 'Longitude'].apply(dms_to_dd)
```

- This block of code applies the `dms_to_dd` function to the **Latitude** and **Longitude** columns, creating new columns **Latitude_dec** and **Longitude_dec** with decimal values.  
- The `.loc[:, 'column_name']` ensures that pandas doesn't trigger warnings while updating the DataFrame.

### 6. Initialize the Nominatim Geolocator

```python
geolocator = Nominatim(user_agent="my-geocoder-app", timeout=10)
```
Initializes the **Nominatim geocoder** from OpenStreetMap, with a custom user agent string (`"my-geocoder-app"`) and a 10-second timeout to avoid early connection failures.

### 7. Define the Reverse Geocoding Function
```python
def get_county(lat, lon):
    if pd.isna(lat) or pd.isna(lon):
        return 'Invalid Coordinates'
    try:
        location = geolocator.reverse((lat, lon), exactly_one=True)
        if location and 'county' in location.raw['address']:
            return location.raw['address']['county']
        return 'County not found'
    except (GeocoderTimedOut, GeocoderUnavailable):
        time.sleep(2)
        try:
            location = geolocator.reverse((lat, lon), exactly_one=True)
            if location and 'county' in location.raw['address']:
                return location.raw['address']['county']
            return 'County not found'
        except (GeocoderTimedOut, GeocoderUnavailable):
            return 'Timed out'
```

- This function reverse geocodes the latitude and longitude using the **Nominatim API** to find the county.  
- It handles invalid coordinates by checking if either latitude or longitude is **NaN**.  
- In case of a timeout or unavailable service, it waits for 2 seconds and retries the request once.  
- If the county is found in the result, it returns the county name. Otherwise, it returns `'County not found'` or `'Timed out'`.

### 8. Apply Reverse Geocoding to the DataFrame
```python
latlongs.loc[:, 'County'] = latlongs.apply(lambda row: get_county(row['Latitude_dec'], row['Longitude_dec']), axis=1)
```
This applies the `get_county()` function to each row of the DataFrame, reverse geocoding the decimal latitude and longitude to obtain the county.  
The `County` column is added to the `latlongs` DataFrame.

### 9. Add the newly added county column to the original dataframe
```python
locations['County'] = latlongs['County']
```
The above code adss the newly generated `County` column from the `latlongs` DataFrame back into the original `locations` DataFrame.

### 10. Save the updated dataframe to a desired location
```python
locations.to_excel('SL0202_Replacement_Locations_with_County.xlsx', index=False)
```
- The code saves the updated DataFrame to a new Excel fil  
- The `index=False` argument ensures that the row index isn't saved in the Excel file.













