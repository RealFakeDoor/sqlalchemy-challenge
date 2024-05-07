# sqlalchemy-challenge
sql-alchemy Challenge.

%matplotlib inline
from matplotlib import style
style.use('fivethirtyeight')
import matplotlib.pyplot as plt

import numpy as np
import pandas as pd
import datetime as dt

# Create engine using the `hawaii.sqlite` database file
engine = create_engine("sqlite:///Resources/hawaii.sqlite")

# Declare a Base using `automap_base()`
Base = automap_base()

# Use the Base class to reflect the database tables
Base.prepare(engine, reflect=True)

# Print all of the classes mapped to the Base
Base.classes.keys()

# Assign the measurement class to a variable called `Measurement`
Measurement = Base.classes.measurement

# Assign the station class to a variable called `Station`
Station = Base.classes.station

# Create a session
session = Session(engine)

# Find the most recent date in the data set.
session.query(func.max(Measurement.date)).all()[0]

# Design a query to retrieve the last 12 months of precipitation data and plot the results.
# Starting from the most recent data point in the database.
most_recent_date = session.query(func.max(Measurement.date)).scalar()

# Calculate the date one year from the last date in data set.
one_year_ago = pd.to_datetime(most_recent_date) - pd.DateOffset(years=1)
one_year_ago = one_year_ago.strftime("%Y-%m-%d")

# Perform a query to retrieve the data and precipitation scores
precipitation_data = session.query(Measurement.date, Measurement.prcp)\
    .filter(Measurement.date >= one_year_ago)\
    .order_by(Measurement.date).all()

# Save the query results as a Pandas DataFrame. Explicitly set the column names
df = pd.DataFrame(precipitation_data, columns=['Date', 'Precipitation'])

# Sort the dataframe by date
df = df.sort_values('Date')

# Use Pandas Plotting with Matplotlib to plot the data:
plt.figure(figsize=(9, 8))
plt.plot(df['Date'], df['Precipitation'], label='precipitation')
plt.xlabel('Date', fontsize=25)
plt.ylabel('mm', fontsize=25)

# formatting to display only 5-dates.
num_dates = 5
step = len(df) // (num_dates - 1)
selected_dates = df['Date'][::step]

plt.xticks(selected_dates, rotation=90, fontsize=20)
plt.yticks(fontsize = 20)
plt.legend(loc='upper center', fontsize=18)
plt.tight_layout()
plt.show()

# Use Pandas to calculate the summary statistics for the precipitation data
summary_statistics = df['Precipitation'].describe()
summary_statistics = pd.DataFrame(summary_statistics)
summary_statistics

# Design a query to calculate the total number of stations in the dataset
total_stations = session.query(func.count(Station.id)).all()
print(total_stations)

# Design a query to find the most active stations (i.e. which stations have the most rows?)
# List the stations and their counts in descending order.
most_active_stations = session.query(Measurement.station, func.count(Measurement.id))\
    .group_by(Measurement.station)\
    .order_by(desc(func.count(Measurement.id)))\
    .all()

most_active_stations

# Using the most active station id from the previous query, calculate the lowest, highest, and average temperature.
most_active_station_id = most_active_stations[0][0]

temperature_stats = session.query(func.min(Measurement.tobs), func.max(Measurement.tobs), func.avg(Measurement.tobs))\
    .filter(Measurement.station == most_active_station_id)\
    .all()

temperature_stats

# Using the most active station id
# Query the last 12 months of temperature observation data for this station and plot the results as a histogram
temperature_data = session.query(Measurement.tobs)\
    .filter(Measurement.station == most_active_station_id)\
    .filter(Measurement.date >= one_year_ago)\
    .all()

# Extract temperature observations from the query results
temperatures = [temp[0] for temp in temperature_data]

# Create a histogram plot
plt.figure(figsize=(10, 8))
plt.hist(temperatures, bins=12, color='dodgerblue', label='tobs')
plt.xlabel('Temperature',fontsize = 22)
plt.ylabel('Frequency', fontsize=22)
plt.grid(True)
plt.legend(loc='upper right', fontsize=20)
plt.tight_layout()
plt.show()

# Close Session
session.close()


APP.PY

# Import the dependencies.
from flask import Flask, jsonify

import datetime as dt

# Python SQL toolkit and Object Relational Mapper
import sqlalchemy
from sqlalchemy.ext.automap import automap_base
from sqlalchemy.orm import Session
from sqlalchemy import create_engine, func, desc

#################################################
# Database Setup
#################################################

# Create engine using the `hawaii.sqlite` database file
engine = create_engine("sqlite:///Resources/hawaii.sqlite")

# Declare a Base using `automap_base()`
Base = automap_base()

# Use the Base class to reflect the database tables
Base.prepare(autoload_with=engine)

# Assign the measurement class to a variable called `Measurement` and
Measurement = Base.classes.measurement

# the station class to a variable called `Station`
Station = Base.classes.station

# Create a session
session = Session(engine)

#################################################
# Flask Setup
#################################################
app = Flask(__name__)
#################################################
# Flask Routes
#################################################

# Define what to do when a user hits the index route.
@app.route("/")
def home():
    """List all available api routes."""
    return (
        f"Available Routes:<br/>"
        f"/api/v1.0/precipitation<br/>"
        f"/api/v1.0/stations<br/>"
        f"/api/v1.0/tobs<br/>"
        f"/api/v1.0/start<br/>"
        f"/api/v1.0/start/end<br/>"
    )

# Returns the JSON representaion of your dictionary
# Convert the query results to a dictionary by using date as the key and prcp as the value.
# Return the JSON representation of your dictionary:
@app.route("/api/v1.0/precipitation")
def precipitation():
    session = Session(engine)
    # Query precipitation data
    results = session.query(Measurement.date, Measurement.prcp).all()
    
    # Convert the results to a dictionary with date as key and prcp as value
    precipitation_dict = {}
    for result in results:
        date = result[0]
        prcp = result[1]
        precipitation_dict[date] = prcp
    
    # Return the precipitation dictionary in JSON format
    return jsonify(precipitation_dict)

# Returns a JSON list of stations from the dataset:
@app.route("/api/v1.0/stations")
def stations():
    session = Session(engine)
    # Query the stations
    station_list = []
    stations = session.query(Station.station, Station.name).all()
    for station in stations:
        station_dict = {}
        station_dict["station"] = station[0]
        station_dict["name"] = station[1]
        station_list.append(station_dict)

    # Convert the list of dictionaries into JSON format and return
    return jsonify(station_list)


# Query the dates and temperature observations of the most-active station for the previous year of data.
# Return a JSON list of temperature observations for the previous year.
@app.route("/api/v1.0/tobs")
def tobs():
    session = Session(engine)
    # Find the most active station
    most_active_station = session.query(Measurement.station, func.count(Measurement.station))\
                            .group_by(Measurement.station).order_by(func.count(Measurement.station).desc())\
                            .first()[0]

    # Determine the date range for the previous year
    most_recent_date = session.query(func.max(Measurement.date)).scalar()
    previous_year_date = dt.datetime.strptime(most_recent_date, '%Y-%m-%d') - dt.timedelta(days=365)
    
    # Query temperature observations for the most active station within the previous year
    results = session.query(Measurement.date, Measurement.tobs)\
                    .filter(Measurement.station == most_active_station)\
                    .filter(Measurement.date >= previous_year_date).all()
    
    # Convert the results to a list of dictionaries
    tobs_list = []
    for result in results:
        tobs_dict = {}
        tobs_dict["date"] = result[0]
        tobs_dict["temperature"] = result[1]
        tobs_list.append(tobs_dict)
    
    # Return the temperature observations in JSON format
    return jsonify(tobs_list)


# Return a JSON list of the minimum temperature, the average temperature, and the maximum temperature for a specified start or start-end range:


# For a specified start, calculate TMIN, TAVG, and TMAX for all the dates greater than or equal to the start date.
@app.route("/api/v1.0/<start>")
def start_date(start):
    # Query minimum, average, and maximum temperatures for dates greater than or equal to the start date.
    session = Session(engine)
    results = session.query(func.min(Measurement.tobs), func.avg(Measurement.tobs), func.max(Measurement.tobs)) \
                    .filter(Measurement.date >= start) \
                    .all()
    
    # Extract the results
    TMIN = results[0][0]
    TAVG = results[0][1]
    TMAX = results[0][2]
    
    # Create a dictionary to hold the results
    temp_stats = {
        "TMIN": TMIN,
        "TAVG": TAVG,
        "TMAX": TMAX
    }
    
    # Return the results in JSON format
    return jsonify(temp_stats)



# For a specified start date and end date, calculate TMIN, TAVG, and TMAX for the dates from the start date to the end date, inclusive.
@app.route("/api/v1.0/<start>/<end>")
def start_end_date(start, end):
# Query minimum, average, and maximum temperatures for dates between the start date and end date (inclusive)
    session = Session(engine)
    results = session.query(func.min(Measurement.tobs), func.avg(Measurement.tobs), func.max(Measurement.tobs)) \
                    .filter(Measurement.date >= start) \
                    .filter(Measurement.date <= end) \
                    .all()
    
    # Extract the results
    TMIN = results[0][0]
    TAVG = results[0][1]
    TMAX = results[0][2]
    
    # Create a dictionary to hold the results
    temp_stats = {
        "TMIN": TMIN,
        "TAVG": TAVG,
        "TMAX": TMAX
    }
    
    # Return the results in JSON format
    return jsonify(temp_stats)

if __name__ == '__main__':
    app.run(debug=True)