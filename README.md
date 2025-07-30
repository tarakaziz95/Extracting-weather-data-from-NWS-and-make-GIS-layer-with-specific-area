# Extracting-weather-data-from-NWS-and-make-GIS-layer-with-specific-area
For utility substations, weather data has been grabbed from NWS and distrute them by substations. You will need ArcGIS Enterprise access. 



import arcpy
import requests
import traceback
from datetime import datetime, timezone

# --- Configuration ---
layer_url = "Hosted Feature layer"
input_sr = arcpy.SpatialReference(2274)  # Replace with your local projection if needed
output_sr = arcpy.SpatialReference(4326)  # WGS 84

# Load FeatureLayer
arcpy.env.overwriteOutput = True
layer = arcpy.FeatureSet()
layer.load(layer_url)

# Fields to update (correct field names)
fields = ["SHAPE@", "temperature", "wind", "humidity", "pressure", 
          "precipitation", "lightning_strikes", "weather_timestamp"]

with arcpy.da.UpdateCursor(layer, fields) as cursor:
    for idx, row in enumerate(cursor):
        try:
            original_geom = row[0]
            geom_projected = original_geom.projectAs(output_sr)
            lon, lat = geom_projected.centroid.X, geom_projected.centroid.Y
            print(f"[{idx}] 🌐 Reprojected to (Lat: {lat}, Lon: {lon})")

            # Step 1: Get forecast & station metadata
            meta_url = f"https://api.weather.gov/points/{lat},{lon}"
            meta_resp = requests.get(meta_url, timeout=10)
            meta = meta_resp.json()
            forecast_url = meta['properties'].get('forecastHourly')
            stations_url = meta['properties'].get('observationStations')
            if not forecast_url or not stations_url:
                print(f"[{idx}] ⚠️ No forecast/station data available.")
                continue

            # Step 2: Get hourly forecast
            forecast = requests.get(forecast_url, timeout=10).json()
            first_hour = forecast['properties']['periods'][0]
            temperature = first_hour['temperature']
            wind_speed_raw = first_hour.get('windSpeed', '0 mph')
            wind_gust = float(wind_speed_raw.split()[0]) if wind_speed_raw else 0
            humidity = first_hour.get('relativeHumidity', {}).get('value')

            # Step 3: Get latest observations
            stations = requests.get(stations_url, timeout=10).json()
            station_id = stations['observationStations'][0].split('/')[-1]
            obs_url = f"https://api.weather.gov/stations/{station_id}/observations/latest"
            obs = requests.get(obs_url, timeout=10).json()
            obs_data = obs.get('properties', {})
            pressure_pa = obs_data.get("seaLevelPressure", {}).get("value")
            pressure = pressure_pa / 100 if pressure_pa else None
            precip_mm = obs_data.get("precipitationLastHour", {}).get("value")

            # Step 4: Set updated attributes
            lightning = 0
            timestamp = datetime.now(timezone.utc).strftime("%Y-%m-%d %H:%M:%S")

            row[1] = temperature
            row[2] = wind_gust
            row[3] = humidity
            row[4] = pressure
            row[5] = precip_mm
            row[6] = lightning
            row[7] = timestamp

            cursor.updateRow(row)
            print(f"[{idx}] ✅ Weather updated.")

        except Exception as e:
            print(f"[{idx}] ❌ Error: {traceback.format_exc()}")

print("🎯 Weather update process completed.")
