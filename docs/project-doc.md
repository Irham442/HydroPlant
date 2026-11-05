# HydroPlant
## Project Technical Documentation

## 1. Project Overview

HydroPlant is an IoT-based smart plant monitoring and automatic irrigation system designed to monitor soil moisture and environmental conditions, then determine the appropriate watering volume using a Simulated Annealing optimization algorithm.

The system integrates an ESP32 microcontroller, a Python-based backend, a Supabase database, and a mobile application.

The main objective is to automate plant watering based on soil moisture conditions while providing users with monitoring information, plant status, and irrigation history.

### Main Features

- Real-time soil moisture and environmental monitoring.
- Automatic irrigation based on soil moisture targets.
- Water volume optimization using Simulated Annealing.
- Automatic nutrient scheduling.
- Plant condition classification using a rule-based emotion system.
- Water tank availability monitoring.
- OLED display for local monitoring.
- Mobile dashboard and monitoring history.
- Plant profile and monitoring management.

## 2. System Architecture

HydroPlant consists of four main components:

1. **ESP32 Firmware:** Reads sensor data, communicates with the backend, and controls irrigation pumps.
2. **Backend API:** Processes sensor readings, determines plant conditions, and calculates watering decisions.
3. **Supabase:** Stores plant profiles, user plant configurations, and monitoring records.
4. **Mobile Application:** Displays monitoring information and allows users to manage plant configurations.

### Communication Flow

The system uses the following communication architecture:

- ESP32 communicates with the FastAPI backend through HTTP requests.
- FastAPI communicates with Supabase to retrieve plant configurations and store monitoring records.
- The mobile application communicates directly with Supabase using the Supabase JavaScript client.

The mobile application does not communicate with FastAPI for its current implemented features.

### Main Data Flow

1. ESP32 reads soil moisture, temperature, humidity, and water tank availability.
2. ESP32 sends sensor readings to the FastAPI `/analyze` endpoint.
3. The backend retrieves the corresponding plant configuration from Supabase.
4. The backend determines the plant condition and calculates watering requirements.
5. The backend returns the analysis results to ESP32.
6. ESP32 activates the appropriate pump based on the returned decision.
7. After watering, ESP32 sends another reading using the `after_watering` phase.
8. Monitoring records are stored in Supabase.
9. The mobile application retrieves stored information directly from Supabase.

## 3. Hardware and Firmware

### 3.1 Hardware Components

The firmware is developed using C++ with the Arduino framework.

The system uses the following hardware components:

| Component | Function | ESP32 Pin |
|---|---|---|
| ESP32 | Main microcontroller | - |
| Soil Moisture Sensor | Measures soil moisture | GPIO 34 |
| DHT22 | Measures temperature and air humidity | GPIO 4 |
| Float Sensor | Detects water tank availability | GPIO 32 |
| Water Pump Relay | Controls the irrigation pump | GPIO 26 |
| Nutrient Pump Relay | Controls the nutrient pump | GPIO 27 |
| OLED Display | Displays monitoring information | SDA 21, SCL 22 |

The OLED display uses an SSD1306 driver with a resolution of 128 × 64 pixels and I2C address `0x3C`.

### 3.2 Sensor Monitoring

ESP32 periodically reads the connected sensors.

**Soil Moisture**

The analog soil sensor reading is converted into a percentage using calibration values:

- Dry soil: 2580
- Wet soil: 704

The resulting value is constrained between 0% and 100%.

**Temperature and Humidity**

The DHT22 sensor provides temperature and air humidity readings.

If the sensor returns an invalid reading, the firmware displays an error and skips the current monitoring request.

**Water Availability**

The float sensor detects whether water is available in the tank.

The firmware interprets a LOW digital reading as water being available.

### 3.3 Backend Communication

ESP32 sends an HTTP POST request containing the following JSON data:

- `sensor_number`
- `soil`
- `temperature`
- `humidity`
- `reading_phase`
- `water_available`

The request is sent to the backend `/analyze` endpoint.

The firmware processes the JSON response to retrieve:

- Device connection status.
- Watering decision.
- Water volume and execution duration.
- Nutrient decision and execution duration.
- Plant emotion and condition status.

### 3.4 Irrigation Control

The firmware controls two pumps through relay modules.

- Water pump: GPIO 26.
- Nutrient pump: GPIO 27.

Both relays use active-low logic:

- LOW: Relay ON.
- HIGH: Relay OFF.

The pumps operate according to the execution duration returned by the backend.

When watering is required, ESP32 activates the water pump. After the watering process, it waits approximately three seconds and sends another sensor reading using the `after_watering` phase.

This second phase prevents the backend from issuing another watering decision during the immediate post-watering reading.

Nutrient pump execution is restricted to the `normal_monitoring` phase.

### 3.5 OLED Display

The OLED display provides local monitoring information, including:

- Soil moisture.
- Temperature.
- Air humidity.
- Plant condition.
- System status.

It also displays messages related to connectivity, watering operations, and water tank availability.

## 4. Backend System

### 4.1 Technology Stack

The backend is developed using:

- Python
- FastAPI
- Pydantic
- Supabase Python Client
- Python Dotenv

The backend exposes an HTTP API for communication with ESP32.

### 4.2 Main Components

The backend consists of three primary Python files.

| File | Responsibility |
|---|---|
| `main.py` | API endpoints, Supabase integration, sensor processing, and irrigation decisions |
| `sa_irrigation.py` | Simulated Annealing optimization algorithm |
| `emotion.py` | Rule-based plant condition classification |

### 4.3 API Endpoint

**POST `/analyze`**

This endpoint receives sensor readings from ESP32 and processes them to determine the appropriate irrigation decision.

The request contains:

| Parameter | Description |
|---|---|
| `sensor_number` | Identifies the connected sensor |
| `soil` | Current soil moisture percentage |
| `temperature` | Current temperature |
| `humidity` | Current air humidity |
| `reading_phase` | Monitoring phase |
| `water_available` | Water tank availability |

The supported monitoring phases are:

- `normal_monitoring`
- `after_watering`

The endpoint performs the following operations:

1. Validates the monitoring phase.
2. Retrieves the active plant configuration from `user_plants`.
3. Retrieves the corresponding plant profile.
4. Determines the plant condition.
5. Evaluates watering requirements.
6. Runs the Simulated Annealing algorithm when watering is required and water is available.
7. Checks the nutrient schedule.
8. Stores monitoring information in Supabase.
9. Returns the analysis results to ESP32.

If no active plant is associated with the sensor, the backend returns a response indicating that the device is not connected to an active plant.

### 4.4 Plant Emotion System

The `emotion.py` module uses a rule-based classification system to represent plant conditions based on soil moisture and the plant's target moisture level.

The classification consists of four conditions:

| Emotion | Condition |
|---|---|
| Sad (`T_T`) | Severe water deficiency |
| Thirsty (`-_-`) | Soil moisture is below the target |
| Happy (`^_^`) | Soil moisture is within the defined optimal range |
| Anxious (`o_o`) | Soil moisture is excessively high |

The classification is determined using the difference between current soil moisture and target soil moisture.

This is a rule-based representation, not a machine learning or AI emotion recognition model.

### 4.5 Nutrient Scheduling

The backend checks nutrient requirements using the plant profile configuration.

The scheduling process considers:

- `nutrient_interval_days`
- `nutrient_volume_ml`
- `last_nutrient_at`

The backend determines whether the nutrient interval has elapsed since the previous recorded nutrient application.

Nutrient execution duration is calculated using the configured pump flow rate:

`WATERING_FLOW_RATE_ML_PER_SEC = 1.83`

The calculated duration is rounded up to the nearest second.

## 5. Simulated Annealing Optimization

### 5.1 Overview

HydroPlant uses Simulated Annealing (SA) to determine a suitable watering volume based on current soil moisture, target soil moisture, temperature, and air humidity.

The algorithm searches for a watering volume that minimizes the difference between predicted final soil moisture and the target value while considering water usage.

### 5.2 Calibration Data

The algorithm uses calibration data based on Sirih Gading plant testing.

| Water Volume (mL) | Soil Moisture (%) |
|---:|---:|
| 0.00 | 51 |
| 5.49 | 53 |
| 10.98 | 55 |
| 16.47 | 58 |
| 21.96 | 61 |
| 27.45 | 63 |
| 32.94 | 65 |
| 38.43 | 66 |
| 43.92 | 70 |

The calibration dataset is used to estimate the expected increase in soil moisture based on the supplied water volume.

### 5.3 Environmental Factor

The algorithm adjusts its predicted soil moisture gain using temperature and air humidity.

The environmental factor starts at 1.0.

Temperature adjustments:

- Temperature ≥ 32°C: subtract 0.15.
- Temperature ≥ 30°C: subtract 0.10.

Humidity adjustments:

- Humidity ≤ 45%: subtract 0.15.
- Humidity ≤ 55%: subtract 0.10.

The resulting factor has a minimum value of 0.65.

This factor is a heuristic adjustment used by the optimization model.

### 5.4 Cost Function

The cost function evaluates the difference between predicted final soil moisture and the target soil moisture, while also applying a penalty for water usage.

The implemented formula is:

`Cost = |Target Soil - Predicted Final Soil| + (Water Volume × 0.03)`

A lower cost represents a better candidate solution according to the model.

### 5.5 Simulated Annealing Parameters

The algorithm uses the following parameters:

| Parameter | Value |
|---|---:|
| Initial Temperature | 100 |
| Cooling Rate | 0.95 |
| Minimum Temperature | 0.1 |
| Initial Volume Range | 0–60 mL |
| Neighbor Adjustment | ±5 mL |

The algorithm generates candidate watering volumes and evaluates them using the cost function.

A candidate with a lower cost is accepted. A candidate with a higher cost may also be accepted according to the simulated annealing probability, allowing the search to explore different solutions.

The best candidate found during the search is returned to the backend.

### 5.6 Optimization Output

The algorithm returns information including:

- Current soil moisture.
- Target soil moisture.
- Temperature.
- Air humidity.
- Environmental factor.
- Recommended watering volume.
- Predicted final soil moisture.
- Cost value.

The backend converts the recommended water volume into pump execution duration using the configured flow rate.

## 6. Mobile Application

### 6.1 Technology Stack

The mobile application is developed using:

- React Native
- Expo SDK 54
- TypeScript
- Expo Router
- Supabase JavaScript Client
- React Navigation

The application is configured to support Android, iOS, and Web platforms.

### 6.2 Supabase Integration

The mobile application uses a shared Supabase client defined in:

`lib/supabase.ts`

The client is initialized using two environment variables:

- `EXPO_PUBLIC_SUPABASE_URL`
- `EXPO_PUBLIC_SUPABASE_KEY`

The application communicates directly with Supabase to retrieve and update plant information and monitoring records.

The `.env` file is excluded from the repository. Users cloning the project must provide their own Supabase project configuration.

### 6.3 Dashboard

File: `app/(tabs)/index.tsx`

The dashboard provides an overview of the HydroPlant system.

Main features include:

- Device connection status based on the latest monitoring record.
- Number of registered plants.
- Last monitoring update.
- List of monitored plants.
- Plant registration.

The dashboard retrieves information from:

- `plant_profiles`
- `user_plants`
- `monitoring_logs`

Device status is determined using the timestamp of the latest monitoring record:

- Online: Latest update is no more than 35 minutes old.
- Offline: Latest update is older than 35 minutes.
- Not connected: No monitoring record is available.

The dashboard also allows users to register a plant by selecting a plant profile and providing a plant name and location.

### 6.4 Monitoring History

File: `app/(tabs)/history.tsx`

The history page displays monitoring records stored in `monitoring_logs`.

Users can select a date to filter the displayed records.

The page displays:

- Plant name.
- Monitoring timestamp.
- Monitoring phase.
- Plant emotion and condition status.
- Soil moisture and target moisture.
- Temperature and air humidity.
- Watering decision.
- Water volume and execution duration.
- Water availability.
- Predicted final soil moisture.
- Simulated Annealing cost.
- Pump flow rate.

Records are ordered from newest to oldest.

Both `normal_monitoring` and `after_watering` records are included.

### 6.5 Plant Profiles

File: `app/(tabs)/profile.tsx`

The plant profile page provides a catalog of plant types supported by the system.

The information is retrieved from `plant_profiles`.

Displayed information includes:

- Common name.
- Scientific name.
- Plant type.
- Target soil moisture.
- Nutrient interval.
- Watering-related configuration fields.
- Pump flow rate.
- Plant care recommendations.

This page serves as a reference for plant characteristics used by the HydroPlant system.

### 6.6 Plant Detail

File: `app/plant-detail.tsx`

The plant detail page displays information about an individual registered plant.

The page retrieves data from:

- `user_plants`
- `plant_profiles`
- `monitoring_logs`

The latest monitoring record is selected using the corresponding `user_plant_id`.

Displayed information includes:

- Plant identity and profile.
- Sensor number.
- Current soil moisture.
- Temperature and humidity.
- Target soil moisture.
- Plant emotion and condition.
- Latest watering decision.
- Recommended water volume and execution duration.
- Predicted soil moisture.
- Simulated Annealing cost.
- Water availability.
- Latest monitoring timestamp.
- Nutrient configuration and plant location.

Users can also edit plant information, start monitoring, or stop monitoring.

### 6.7 User Authentication

The current application does not implement login or registration.

HydroPlant is designed for a single-owner monitoring environment. User authentication is therefore outside the current project scope.

The application uses Supabase project configuration credentials to access the database.

## 7. Database Structure

HydroPlant uses Supabase as its database and backend service.

The source code confirms the use of the following tables:

### 7.1 `plant_profiles`

Stores plant reference information and configuration.

Relevant fields include:

- `id`
- `plant_type`
- `common_name`
- `scientific_name`
- `target_soil`
- `nutrient_interval_days`
- `nutrient_volume_ml`
- `description`

### 7.2 `user_plants`

Stores plant records registered for monitoring.

Relevant fields include:

- `id`
- `profile_id`
- `plant_name`
- `sensor_number`
- `is_active`
- `location`
- `notes`
- `last_nutrient_at`

The table connects registered plants to their corresponding plant profiles.

### 7.3 `monitoring_logs`

Stores monitoring readings and system decisions.

Relevant fields include:

- `id`
- `user_plant_id`
- `plant_type`
- `soil`
- `temperature`
- `humidity`
- `target_soil`
- `needs_watering`
- `watering_volume_ml`
- `watering_duration_seconds`
- `watering_flow_rate_ml_per_sec`
- `water_available`
- `emotion`
- `emoji`
- `status`
- `reading_phase`
- `predicted_final_soil`
- `cost`
- `created_at`

These records are used by the mobile application to display monitoring history and the latest plant condition.

## 8. Project Configuration

### 8.1 Backend Environment Variables

The backend requires the following environment variables:

| Variable | Description |
|---|---|
| `SUPABASE_URL` | Supabase project URL |
| `SUPABASE_KEY` | Supabase API key used by the backend |

These values are loaded using Python Dotenv.

### 8.2 Mobile Environment Variables

The mobile application requires:

| Variable | Description |
|---|---|
| `EXPO_PUBLIC_SUPABASE_URL` | Supabase project URL |
| `EXPO_PUBLIC_SUPABASE_KEY` | Supabase publishable key or legacy anon key |

The `.env` file must not be committed to the repository.

The `EXPO_PUBLIC_` prefix indicates that these variables are exposed to the application runtime. Do not use a Supabase `service_role` key in the mobile application.

### 8.3 ESP32 Configuration

The firmware requires configuration for:

- Wi-Fi SSID.
- Wi-Fi password.
- Backend server address.

The firmware source contains placeholder values for these settings. They must be replaced with the appropriate local configuration before deployment.

Real credentials should not be committed to the public repository.

## 9. Current Limitations

The following limitations reflect the current implementation.

### Hardware and Monitoring

- The firmware currently uses sensor number 1.
- The available sensor configuration in the mobile application is also limited to sensor number 1.
- Monitoring depends on Wi-Fi connectivity and backend availability.
- Soil moisture readings depend on the calibration values configured in the firmware.

### Backend and Optimization

- Simulated Annealing calibration data is based on Sirih Gading testing.
- The optimization model may not generalize equally to every supported plant type.
- Environmental adjustments use heuristic factors rather than a direct physical evaporation model.
- Randomized initialization and candidate selection can produce different optimization results for identical inputs.
- Nutrient scheduling updates `last_nutrient_at` when the backend makes the decision, before the firmware confirms physical pump execution.

### Mobile Application

- No login or registration system is implemented.
- The mobile application connects directly to Supabase.
- Device status is inferred from the latest monitoring timestamp rather than a direct device heartbeat.
- Manual watering is not currently implemented.
- Plant removal from monitoring uses an inactive status rather than permanently deleting the database record.
- The current sensor selection and assignment logic is limited to sensor number 1.

### Database and Security

- Supabase project configuration must be provided separately through environment variables.
- The repository does not contain the owner's `.env` configuration.
- Database access policies, including Row Level Security (RLS), are not documented here because their actual configuration has not been verified.

## 10. Project Directory Structure

The consolidated repository uses the following monorepo structure:

```text
HydroPlant/
├── README.md
├── .gitignore
│
├── backend/
│   ├── main.py
│   ├── emotion.py
│   ├── sa_irrigation.py
│   └── .env.example
│
├── firmware/
│   └── esp32/
│       └── ESP32-Dev-Kit1-Hydro-plant.ino
│
├── mobile/
│   ├── app/
│   ├── components/
│   ├── constants/
│   ├── hooks/
│   ├── lib/
│   ├── assets/
│   ├── package.json
│   └── .env.example
│
└── docs/
    ├── architecture/
    │   └── system-architecture.png
    ├── screenshots/
    │   ├── dashboard.png
    │   └── OLED-plant-status.png
    └── project-doc.md
```

The directory structure represents the consolidated organization of the firmware, backend, and mobile application within one repository.

## 11. Conclusion

HydroPlant integrates IoT hardware, backend processing, database services, and a mobile application to provide an automated plant monitoring and irrigation system.

By combining sensor readings with a Simulated Annealing optimization algorithm, the system attempts to determine an appropriate watering volume based on plant moisture requirements and environmental conditions.

The ESP32 handles physical monitoring and pump execution, the backend processes sensor data and irrigation decisions, Supabase stores system information, and the mobile application provides an interface for monitoring and plant management.

The project demonstrates the integration of IoT, API development, database management, and mobile application development into one connected system.