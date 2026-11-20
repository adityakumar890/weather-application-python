
import tkinter as tk
from tkinter import messagebox
import requests
import threading

# Weather condition descriptions
WEATHER_CODES = {
    0: "Clear Sky",
    1: "Mainly Clear",
    2: "Partly Cloudy",
    3: "Overcast",
    45: "Foggy",
    48: "Depositing Rime Fog",
    51: "Light Drizzle",
    53: "Moderate Drizzle",
    55: "Heavy Drizzle",
    61: "Light Rain",
    63: "Moderate Rain",
    65: "Heavy Rain",
    71: "Light Snow",
    73: "Moderate Snow",
    75: "Heavy Snow",
    80: "Rain Showers",
    81: "Moderate Rain Showers",
    82: "Heavy Rain Showers",
    95: "Thunderstorm",
    96: "Thunderstorm with Hail",
    99: "Severe Thunderstorm"
}


def get_weather(city):
    # Find city coordinates
    geo_url = "https://geocoding-api.open-meteo.com/v1/search"

    geo_response = requests.get(
        geo_url,
        params={
            "name": city,
            "count": 1,
            "language": "en",
            "format": "json"
        },
        timeout=10
    )
    geo_response.raise_for_status()

    geo_data = geo_response.json()

    if not geo_data.get("results"):
        raise ValueError("City not found. Please try another city.")

    place = geo_data["results"][0]
    latitude = place["latitude"]
    longitude = place["longitude"]
    city_name = place["name"]
    country = place.get("country", "")

    # Get current weather
    weather_url = "https://api.open-meteo.com/v1/forecast"

    weather_response = requests.get(
        weather_url,
        params={
            "latitude": latitude,
            "longitude": longitude,
            "current": (
                "temperature_2m,relative_humidity_2m,"
                "apparent_temperature,precipitation,"
                "weather_code,wind_speed_10m"
            ),
            "timezone": "auto"
        },
        timeout=10
    )
    weather_response.raise_for_status()

    weather_data = weather_response.json()
    current = weather_data["current"]

    return {
        "city": city_name,
        "country": country,
        "temperature": current["temperature_2m"],
        "feels_like": current["apparent_temperature"],
        "humidity": current["relative_humidity_2m"],
        "wind": current["wind_speed_10m"],
        "precipitation": current["precipitation"],
        "condition": WEATHER_CODES.get(
            current["weather_code"], "Unknown"
        ),
        "time": current["time"]
    }


def search_weather():
    city = city_entry.get().strip()

    if not city:
        messagebox.showwarning(
            "Input Required",
            "Please enter a city name."
        )
        return

    search_button.config(state="disabled", text="Searching...")
    result_label.config(text="Fetching weather information...")

    threading.Thread(
        target=fetch_weather,
        args=(city,),
        daemon=True
    ).start()


def fetch_weather(city):
    try:
        data = get_weather(city)
        root.after(0, lambda: display_weather(data))

    except requests.RequestException:
        root.after(
            0,
            lambda: show_error(
                "Unable to connect. Check your internet connection."
            )
        )

    except (ValueError, KeyError):
        root.after(
            0,
            lambda: show_error(
                "City not found or weather data unavailable."
            )
        )


def display_weather(data):
    result_label.config(
        text=(
            f"Weather in {data['city']}, {data['country']}\n\n"
            f"Temperature: {data['temperature']} °C\n"
            f"Feels Like: {data['feels_like']} °C\n"
            f"Condition: {data['condition']}\n"
            f"Humidity: {data['humidity']}%\n"
            f"Wind Speed: {data['wind']} km/h\n"
            f"Precipitation: {data['precipitation']} mm\n\n"
            f"Updated: {data['time']}"
        )
    )

    search_button.config(state="normal", text="Search Weather")


def show_error(message):
    result_label.config(text=message)
    search_button.config(state="normal", text="Search Weather")


# Main application window
root = tk.Tk()
root.title("Weather Application")
root.geometry("450x520")
root.resizable(False, False)
root.configure(bg="#EAF4FF")

# Application heading
heading = tk.Label(
    root,
    text="WEATHER APPLICATION",
    font=("Arial", 19, "bold"),
    bg="#EAF4FF",
    fg="#123B65"
)
heading.pack(pady=25)

# City input
city_entry = tk.Entry(
    root,
    font=("Arial", 14),
    width=25,
    justify="center"
)
city_entry.pack(pady=10)
city_entry.insert(0, "Enter City Name")

# Search button
search_button = tk.Button(
    root,
    text="Search Weather",
    font=("Arial", 12, "bold"),
    bg="#1677D2",
    fg="white",
    command=search_weather,
    padx=15,
    pady=8
)
search_button.pack(pady=15)

# Weather result
result_label = tk.Label(
    root,
    text="Search for a city to see its weather.",
    font=("Arial", 12),
    bg="#EAF4FF",
    fg="#123B65",
    justify="center",
    wraplength=400
)
result_label.pack(pady=25)

root.mainloop()
