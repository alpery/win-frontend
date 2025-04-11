<script>
    import {onMount} from "svelte";

    let city = "Istanbul";
    let weatherData = [];
    let loading = false;
    let error = null;
    let dailyForecasts = [];
    let selectedDayIndex = 0;
    let recentSearches = [];
    let showRecentSearches = false;
    let isConnected = true;
    let isRecording = false;
    let triggerWord = "Wetter";

    let days = 5;
    let lang = 'de';
    let recognition = null;
    let isListening = false;
    let voicePrompt = "";
    let inputRef = null;


    async function fetchWeatherData() {

        if (!city.trim()) {
            city = "Please enter a city name";
            return;
        }

        loading = true;
        error = null;

        try {
            console.log(city)
            const response = await fetch(
                `http://localhost:8080/api/weather/${city}`,
            );

            if (!response.ok) {
                new Error(
                    `Failed to fetch weather data: ${response.statusText}`,
                );
            }
            console.log(response)
            weatherData = await response.json();
            processWeatherData();
            console.log(dailyForecasts);  // Füge ein, um die Daten zu prüfen
            if (!recentSearches.includes(city)) {
                recentSearches = [city, ...recentSearches.slice(0, 4)];
                localStorage.setItem(
                    "recentSearches",
                    JSON.stringify(recentSearches),
                );
            }

        } catch (err) {
            console.error("Error fetching weather data:", err);
            error = `Error fetching weather data: ${err.message}`;
            weatherData = [];
            dailyForecasts = [];

        } finally {
            loading = false;
        }
    }


    function processWeatherData() {


        if (!weatherData || weatherData.length === 0) {
            dailyForecasts = [];
            return;
        }

        // Group forecasts by day
        const groupedByDay = {};

        weatherData.forEach((item) => {
            const date = new Date(item.forecastDate);
            const dayKey = date.toISOString().split("T")[0]; // YYYY-MM-DD format

            if (!groupedByDay[dayKey]) {
                groupedByDay[dayKey] = [];
            }

            groupedByDay[dayKey].push(item);
        });

        // Create daily summary for each day
        dailyForecasts = Object.keys(groupedByDay)
            .map((day) => {
                const forecasts = groupedByDay[day];
                const dayData = forecasts[0]; // Use first forecast for basic info
                const currentTemp = dayData.temperature
                // Find min and max temperatures for the day
                const minTemp = Math.min(
                    ...forecasts.map((f) => f.minTemperature),
                );
                const maxTemp = Math.max(
                    ...forecasts.map((f) => f.maxTemperature),
                );

                // Calculate average humidity
                const avgHumidity = Math.round(
                    forecasts.reduce((sum, f) => sum + f.humidity, 0) /
                    forecasts.length,
                );

                // Get the noon forecast if available, otherwise use the first one
                const noonForecast =
                    forecasts.find((f) => {
                        const date = new Date(f.forecastDate);
                        return date.getHours() >= 12 && date.getHours() < 15;
                    }) || dayData;

                // Get weather condition category
                const weatherCondition = getWeatherCondition(
                    noonForecast.description,
                );

                // Filter out forecasts with duplicate formatted times
                const uniqueForecasts = [];
                const seenTimes = new Set();

                // Sort forecasts by time first
                const sortedForecasts = forecasts.sort(
                    (a, b) => new Date(a.forecastDate) - new Date(b.forecastDate)
                );

                // Keep only one forecast per formatted time
                sortedForecasts.forEach(forecast => {
                    const formattedTime = new Date(forecast.forecastDate).toLocaleTimeString("de-DE", {
                        hour: "2-digit",
                        minute: "2-digit",
                    });

                    if (!seenTimes.has(formattedTime)) {
                        seenTimes.add(formattedTime);
                        uniqueForecasts.push(forecast);
                    }
                });

                return {
                    date: new Date(day),
                    city: dayData.city,
                    temperature: currentTemp,
                    minTemperature: minTemp,
                    maxTemperature: maxTemp,
                    avgHumidity: avgHumidity,
                    description: noonForecast.description,
                    iconCode: noonForecast.iconCode,
                    weatherCondition: weatherCondition,
                    forecasts: uniqueForecasts, // Use filtered forecasts without duplicates
                };
            })
            .sort((a, b) => a.date - b.date); // Sort by date

        // Reset selected day to first day
        selectedDayIndex = 0;
        city = dailyForecasts[0].city
    }

    // Get weather condition category
    function getWeatherCondition(description) {
        const desc = description.toLowerCase();
        if (desc.includes("klar") || desc.includes("himmel")) return "clear";

        if (desc.includes("wolke") || desc.includes("bewölkt")) {
            return "cloudy";
        }
        if (desc.includes("regen")) {
            return "rain";
        }
        if (desc.includes("schnee")) {
            return "snow";
        }

        if (desc.includes("gewitter") || desc.includes("sturm")) {
            return "storm";
        }
        if (desc.includes("nebel")) return "fog";
        {
            return "default";
        }
    }


    // Format date to display full date
    function formatDate(date) {
        return new Date(date).toLocaleDateString("de-DE", {
            weekday: "long",
            year: "numeric",
            month: "long",
            day: "numeric",
        });
    }


    // Get weather icon URL
    function getWeatherIconUrl(iconCode) {
        if (iconCode.includes('n')) {
            switch (iconCode) {
                case '01n':
                    return `https://openweathermap.org/img/wn/01d@2x.png`;

                case '02n':
                    return `https://openweathermap.org/img/wn/02d@2x.png`;

                case '03n':
                    return `https://openweathermap.org/img/wn/03d@2x.png`;

                case '04n':
                    return `https://openweathermap.org/img/wn/02d@2x.png`;

                case '10n':
                    return `https://openweathermap.org/img/wn/10d@2x.png`;

            }
        }
        if (iconCode === '04d') {
            return `https://openweathermap.org/img/wn/02d@2x.png`;

        }

        return `https://openweathermap.org/img/wn/${iconCode}@2x.png`;

    }

    // Handle form submission
    function handleSubmit(event) {
        event.preventDefault();
        fetchWeatherData();
        showRecentSearches = false;
    }


    // Get background gradient based on time and weather
    function getBackgroundGradient(forecast) {
        if (!forecast) return "linear-gradient(to bottom, #4b6cb7, #182848)";

        const condition = forecast.weatherCondition;
        const hour = new Date(forecast.forecasts[0].forecastDate).getHours();
        const isDay = hour >= 6 && hour < 20;
        console.log(forecast.weatherCondition);

        if (condition === "clear") {
            return isDay
                ? "linear-gradient(to bottom, #2980b9, #6dd5fa, #ffffff)"
                : "linear-gradient(to bottom, #0f2027, #203a43, #2c5364)";
        } else if (condition === "cloudy") {
            return isDay
                ? "linear-gradient(to bottom, #757f9a, #d7dde8)"
                : "linear-gradient(to bottom, #373b44, #4286f4)";
        } else if (condition === "rain") {
            return "linear-gradient(to bottom, #616161, #9bc5c3)";
        } else if (condition === "snow") {
            return "linear-gradient(to bottom, #e6dada, #274046)";
        } else if (condition === "storm") {
            return "linear-gradient(to bottom, #232526, #414345)";
        } else if (condition === "fog") {
            return "linear-gradient(to bottom, #b79891, #94716b)";
        }

        return "linear-gradient(to bottom, #4b6cb7, #182848)";
    }


    function setupSpeechRecognition() {
        if (typeof window === 'undefined') return;

        try {
            // @ts-ignore
            const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
            if (!SpeechRecognition) {
                console.log("Speech Recognition not supported");
                return;
            }

            recognition = new SpeechRecognition();
            recognition.lang = lang === 'de' ? 'de-DE' : 'en-US';
            recognition.continuous = false;

            recognition.onstart = () => isRecording = true;
            recognition.onend = () => {
                isRecording = false;
                if (isListening) {
                    setTimeout(() => recognition.start(), 300);
                }
            };

            recognition.onresult = (event) => {
                const transcript = event.results[0][0].transcript.toLowerCase();
                console.log("Heard:", transcript);

                if (!transcript.includes(triggerWord.toLowerCase())) return;

                // Extract command after trigger word
                const parts = transcript.split(triggerWord.toLowerCase());
                if (parts.length < 2) return;

                let command = parts[1].trim()
                if (command.endsWith('.')) {
                    command = command.slice(0, -1);
                }
                if (!command) return;
                command = command.replace(/\.$/, '');
                if (!command) return;

                // Parse command
                parseVoiceCommand(command);
            };

            isListening = true;
            recognition.start();
        } catch (e) {
            console.error("Speech recognition error:", e);
        }
    }

    function parseVoiceCommand(command) {
        // Extract city and days from command
        const words = command.trim().split(/\s+/);
        let newDays = 5;
        let cityWords = [];

        for (let i = 0; i < words.length; i++) {
            const num = parseInt(words[i]);
            if (!isNaN(num)) {
                newDays = Math.min(Math.max(num, 1), 5); // Limit between 1-5
            } else {
                cityWords.push(words[i]);
            }
        }

        const newCity = cityWords.join(" ").trim();
        if (newCity) {
            city = newCity;
            days = newDays;
            fetchWeatherData();
        }
    }

    function checkConnection() {
        isConnected = navigator.onLine;
        window.addEventListener('online', () => isConnected = true);
        window.addEventListener('offline', () => isConnected = false);
    }


    // Initialize with a default city and load recent searches
    onMount(() => {
        // Load recent searches from localStorage

        setupSpeechRecognition();
        checkConnection();

        document.getElementsByClassName("city-input").value = (" ")

        // Start with a default city

        fetchWeatherData();

        // Initialize speech recognition

    });


    let drops = Array.from({length: 100});
    let snowflakes = Array.from({length: 100});

    let clouds = Array(30).fill(null); // Example: 10 clouds

    function generateCloudStyle() {
        const left = Math.random() * 120;
        const bottom = Math.random() * 550 + 150;
        const opacity = Math.random() * (0.8 - 0.4) + 0.4;
        const scale = Math.random() * (1 - 0.4) + 0.4;
        const animationDelay = 3; // Math.floor(Math.random() * 19);
        const animationDuration = 50; // Math.random() * (25 - 19) + 19;
        return `
        left: ${left}%;
        bottom: ${bottom}px;
        opacity: ${opacity};
        transform: scale(${scale});
        animationDelay: ${animationDelay};
       animation-duration: ${animationDuration}s;
    `;
    }


</script>
<main>
    {#if dailyForecasts.length > 0}

        <div class="weather-animation">

            {#if dailyForecasts.length > 0 && dailyForecasts[selectedDayIndex].weatherCondition === 'rain'}
                <div class="rain-animation">
                    {#each drops as _, i}
                        <div class="drop"
                             style="left: {Math.random() * 100}%; animation-delay: {Math.random()}s;"></div>
                    {/each}
                </div>
            {/if}
            {#if dailyForecasts.length > 0 && dailyForecasts[selectedDayIndex].weatherCondition === 'snow'}
                <div class="snow-animation">
                    {#each snowflakes as _, i}
                        <div class="snowflake"
                             style="left: {Math.random() * 100}%; animation-delay: {Math.random()}s;"></div>
                    {/each}
                </div>
            {/if}
            {#if dailyForecasts.length > 0 && dailyForecasts[selectedDayIndex].weatherCondition === 'cloudy'}
                <div class="cloudy-animation">
                    {#each clouds as _, i}
                        <div class="cloud" style="{generateCloudStyle()}">

                        </div>
                    {/each}
                </div>
            {/if}
            {#if dailyForecasts.length > 0 && dailyForecasts[selectedDayIndex].weatherCondition === 'fog'}
                <div class="fog-animation">
                    {#each Array.from({length: 5}) as _, i}
                        <div class="fog-layer"
                             style="--fog-opacity: {0.3 - i * 0.03}; --fog-animation-duration: {15 + i * 2}s; --fog-animation-delay: {i * 0.5}s;"></div>
                    {/each}
                </div>
            {/if}
            {#if dailyForecasts.length > 0 && dailyForecasts[selectedDayIndex].weatherCondition === 'storm'}
                <div class="storm-animation">
                    {#each drops as _, i}
                        <div class="drop storm-drop"
                             style="left: {Math.random() * 100}%; animation-delay: {Math.random() * 0.5}s;"></div>
                    {/each}
                    <div class="lightning-container">
                        <div class="lightning"></div>
                        <div class="lightning delayed"></div>
                    </div>
                </div>
            {/if}

        </div>
        <div class="weather-app" style="background: {getBackgroundGradient(dailyForecasts[selectedDayIndex])}">
            <div class="app-header">
                <h1>Wetter Vorhersage</h1>

                <div class="voice-hint">
                    <div class="voice-hint-content">
                        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24"
                             fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
                             stroke-linejoin="round">
                            <path d="M12 1a3 3 0 0 0-3 3v8a3 3 0 0 0 6 0V4a3 3 0 0 0-3-3z"></path>
                            <path d="M19 10v2a7 7 0 0 1-14 0v-2"></path>
                            <line x1="12" y1="19" x2="12" y2="23"></line>
                            <line x1="8" y1="23" x2="16" y2="23"></line>
                        </svg>
                        <span>Neu! Sprachsteuerung verfügbar. Sagen Sie "Wetter" und irgendwelche Stadt</span>
                    </div>
                </div>


                <div class="search-container">
                    <form on:submit={handleSubmit} class="search-form">
                        <div class="input-wrapper {isListening ? 'listening' : ''}">
                            <input bind:value={city} bind:this={inputRef}
                                   placeholder={isListening ? voicePrompt : "Stadt eingeben..."} class="city-input"

                                   readonly="readonly"/>
                        </div>
                    </form>
                </div>
            </div>

            {#if error}
                <div class="error-message">
                    {error}
                </div>
            {/if}

            {#if loading}
                <div class="loading-container">
                    <div class="loading-spinner"></div>
                    <div class="loading-text">Wetterdaten werden geladen...</div>
                </div>

            {:else}


                <div class="today-weather-card">
                    <div class="date">{formatDate(dailyForecasts[0].date)}</div>

                    <div class="weather-info">
                        <div class="temperature-container">
                            <div class="current-temp">Aktuelle
                                Temperatur: {Math.round(dailyForecasts[0].temperature)}
                                °C
                            </div>
                            <div class="min-max">
                                <span class="min">{Math.round(dailyForecasts[0].minTemperature)}°C</span> /
                                <span class="max">{Math.round(dailyForecasts[0].maxTemperature)}°C</span>
                            </div>
                        </div>

                        <div class="weather-icon">
                            <img src={getWeatherIconUrl(dailyForecasts[0].iconCode)}
                                 alt={dailyForecasts[0].description}/>
                            <div class="description">{dailyForecasts[0].description}</div>
                        </div>
                    </div>

                    <div class="details">
                        <div class="detail-item">
                            <span class="label">Luftfeuchtigkeit:</span>
                            <span class="value">{dailyForecasts[0].avgHumidity}%</span>
                        </div>
                    </div>
                </div>

                <div class="next-weather">

                    {#each dailyForecasts.slice(1) as weatherData}

                        <div class="weather-card">
                            <div class="date">{formatDate(weatherData.date)}</div>

                            <div class="weather-info">
                                <div class="temperature-container">

                                    <div class="min-max">
                                        <span class="min">{Math.round(weatherData.minTemperature)}°C</span> /
                                        <span class="max">{Math.round(weatherData.maxTemperature)}°C</span>
                                    </div>
                                </div>

                                <div class="weather-icon">
                                    <img src={getWeatherIconUrl(weatherData.iconCode)}
                                         alt={weatherData.description}/>
                                    <div class="description">{weatherData.description}</div>
                                </div>
                            </div>

                            <div class="details">
                                <div class="detail-item">
                                    <span class="label">Luftfeuchtigkeit:</span>
                                    <span class="value">{weatherData.avgHumidity}%</span>
                                </div>
                            </div>
                        </div>
                    {/each}
                </div>
            {/if}

        </div>

    {:else}


        <div class="weather-app" style="background: {getBackgroundGradient(dailyForecasts[selectedDayIndex])}">
            <div class="app-header">
                <h1>Wetter Vorhersage</h1>

                <div class="voice-hint">
                    <div class="voice-hint-content">
                        <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24"
                             fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"
                             stroke-linejoin="round">
                            <path d="M12 1a3 3 0 0 0-3 3v8a3 3 0 0 0 6 0V4a3 3 0 0 0-3-3z"></path>
                            <path d="M19 10v2a7 7 0 0 1-14 0v-2"></path>
                            <line x1="12" y1="19" x2="12" y2="23"></line>
                            <line x1="8" y1="23" x2="16" y2="23"></line>
                        </svg>
                        <span>Neu! Sprachsteuerung verfügbar. Sagen Sie "Wetter" und irgendwelche Stadt</span>
                    </div>
                </div>


                <div class="search-container">
                    <form on:submit={handleSubmit} class="search-form">
                        <div class="input-wrapper {isListening ? 'listening' : ''}">
                            <input bind:value={city} bind:this={inputRef}
                                   placeholder={isListening ? voicePrompt : "Stadt eingeben..."} class="city-input"

                                   readonly="readonly"/>
                        </div>
                    </form>
                </div>
                <div class="no-data">
                    <svg xmlns="http://www.w3.org/2000/svg" width="64" height="64" viewBox="0 0 24 24" fill="none"
                         stroke="currentColor" stroke-width="1" stroke-linecap="round" stroke-linejoin="round">
                        <path d="M17.5 19H9a7 7 0 1 1 6.71-9h1.79a4.5 4.5 0 1 1 0 9Z"></path>
                    </svg>
                    <p>Keine Wetterdaten verfügbar. Bitte suchen Sie nach einer Stadt.</p>
                </div>
            </div>

        </div>
    {/if}


</main>

<style>

    .max {
        color: red;
    }

    .min {
        color: blue;
    }

    .weather-animation {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        pointer-events: none;
        overflow: hidden;
        z-index: 0; /*animation vor(1) oder hinter(0) der wetter karte */
    }

    .drop {
        position: absolute;
        top: -10%;
        width: 2px;
        height: 20px;
        background: rgba(255, 255, 255, 0.5);
        animation: rainDrop 0.7s linear infinite;
    }

    @keyframes rainDrop {
        0% {
            transform: translateY(0);
            opacity: 1;
        }
        100% {
            transform: translateY(120vh);
            opacity: 0;
        }
    }


    /*Schneeflocken Animation*/
    .snowflake {
        --size: 1vw;
        color: white;
        position: absolute;
        top: -5vh;
        background-color: rgba(255, 255, 255, 0.9);
        width: var(--size);
        height: var(--size);
        border-radius: 50%;
        animation: snow 1s linear infinite;
        animation-delay: -7s;
    }


    @keyframes snow {
        0% {

            top: -10px;
            opacity: 1;
        }
        100% {
            top: 100%;
            opacity: 0.5;
        }
    }


    @keyframes snowfall {
        0% {
            transform: translate3d(var(--left-ini), 0, 0);
        }
        100% {
            transform: translate3d(var(--left-end), 110vh, 0);
        }
    }


    .cloud {
        position: absolute;
        bottom: 500px;
        width: 250px;
        height: 120px;
        background: #fff;
        border-radius: 50%;
        box-shadow: 30px 10px 0 0 #fff,
        60px 15px 0 0 #fff,
        20px -10px 0 0 #fff;
        animation-name: floatCloud;
        animation-timing-function: linear;
        animation-iteration-count: infinite;
        opacity: 0.7;
    }

    @keyframes floatCloud {
        0% {
            transform: translateX(-100px);
            opacity: 0.7;
        }
        100% {
            transform: translateX(90vh);
            opacity: 0.7;
        }
    }

    .fog-animation {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        overflow: hidden;
        pointer-events: none;
    }

    .fog-layer {
        position: absolute;
        width: 200%;
        height: 100%;
        background: linear-gradient(90deg, rgba(255, 255, 255, 0) 0%, rgba(255, 255, 255, var(--fog-opacity)) 25%, rgba(255, 255, 255, var(--fog-opacity)) 75%, rgba(255, 255, 255, 0) 100%);
        animation: fogMove var(--fog-animation-duration) linear infinite;
        animation-delay: var(--fog-animation-delay);
        top: calc(10% + var(--fog-animation-delay) * 10);
        opacity: 0.8;
    }

    @keyframes fogMove {
        0% {
            transform: translateX(-50%);
        }
        100% {
            transform: translateX(0%);
        }
    }

    .storm-drop {
        height: 30px;
        transform: rotate(15deg);
        animation: stormDrop 0.5s linear infinite;

    }

    @keyframes stormDrop {
        0% {
            transform: translateY(0) rotate(15deg);
            opacity: 1;
        }
        100% {
            transform: translateY(120vh) rotate(15deg);
            opacity: 0;
        }
    }

    .lightning-container {
        position: absolute;
        width: 100%;
        height: 100%;
        top: 0;
        left: 0;
        pointer-events: none;
    }

    .lightning {
        position: absolute;
        width: 100%;
        height: 100%;
        background-color: rgba(255, 255, 255, 0);
        opacity: 0;
        animation: lightning 4s infinite;
    }

    .lightning.delayed {
        animation: lightning 7s 3.5s infinite;
    }

    @keyframes lightning {
        0%, 95%, 100% {
            background-color: rgba(255, 255, 255, 0);
            opacity: 0;
        }
        96%, 98% {
            background-color: rgba(255, 255, 255, 0.6);
            opacity: 1;
        }
        97%, 99% {
            background-color: rgba(255, 255, 255, 0);
            opacity: 0;
        }
    }

    :global(body) {
        margin: 0;
        padding: 0;
        transition: background 0.5s ease;
        color: white;
        display: flex;
        flex-wrap: wrap;
        flex-direction: column;
        justify-content: space-around;
        font-family: "Calibri", sans-serif;
    }


    .weather-app {
        min-height: 100vh;
        padding: 20px;
        transition: background 0.5s ease;
        color: white;
        display: flex;
        flex-wrap: wrap;
        flex-direction: column;
        justify-content: space-around;
        align-items: center;
    }


    .app-header {
        display: flex;
        justify-content: center;
        align-items: center;
        margin-bottom: 30px;
        flex-wrap: wrap;
        gap: 20px;
        flex-direction: column;
    }

    .next-weather {

        display: flex;
        flex-wrap: wrap; /* Erlaubt den Umbruch, wenn der Platz nicht ausreicht */
        justify-content: space-evenly; /* Verteilt den Platz gleichmäßig */
        gap: 50px; /* Abstand zwischen den Karten */
        padding: 20px;
    }

    .weather-card {
        font-size: x-large;
        text-align: center;
        box-shadow: 0 4px 30px rgba(0, 0, 0, 0.1);
        background-color: rgba(0, 0, 0, 0.2);
        border-radius: 16px;
        padding: 25px;
        margin-bottom: 25px;
        backdrop-filter: blur(10px);
        transition: transform 0.2s;
    }

    .today-weather-card {
        width: 90%;
        font-size: x-large;
        text-align: center;
        box-shadow: 0 4px 30px rgba(0, 0, 0, 0.1);
        background-color: rgba(0, 0, 0, 0.2);
        border-radius: 16px;
        padding: 25px;
        margin-bottom: 25px;
        backdrop-filter: blur(10px);
        transition: transform 0.2s;

    }

    .current-temp {
        margin-top: 10px;
        justify-self: center;
    }

    .today-weather-card:hover {
        transform: translateY(-5px);
    }


    h1 {
        margin: 0;
        font-size: 4rem;
        font-weight: 700;
        text-shadow: 1px 1px 3px rgba(0, 0, 0, 0.2);
        justify-content: center;
        text-align: center;
    }

    h2 {
        margin: 0;
        font-size: 1.8rem;
        font-weight: 600;
        text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.2);
    }

    h3 {
        margin: 20px 0 15px;
        font-size: 1.3rem;
        font-weight: 600;
        text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.2);
    }

    .search-container {
        position: relative;
    }

    .search-form {
        display: flex;
        gap: 10px;
    }

    .input-wrapper {
        position: relative;
    }

    .city-input {
        font-size: 3rem;
        padding: 12px 15px;
        border: none;
        border-radius: 25px;
        width: 450px;
        background-color: rgba(255, 255, 255, 0.2);
        color: white;
        backdrop-filter: blur(10px);
        transition: all 0.3s ease;
        text-align: center;
    }

    .city-input::placeholder {
        color: rgba(255, 255, 255, 0.7);
    }

    .city-input:focus {
        outline: none;
        background-color: rgba(255, 255, 255, 0.3);
        box-shadow: 0 0 0 2px rgba(255, 255, 255, 0.5);
    }


    @keyframes pulse {
        0% {
            box-shadow: 0 0 0 0 rgba(255, 100, 100, 0.5);
        }
        70% {
            box-shadow: 0 0 0 10px rgba(255, 100, 100, 0);
        }
        100% {
            box-shadow: 0 0 0 0 rgba(255, 100, 100, 0);
        }
    }

    .voice-hint {
        font-size: larger;
        background-color: rgba(255, 255, 255, 0.2);
        border-radius: 12px;
        padding: 15px;
        margin-bottom: 15px;
        display: flex;
        align-items: center;
        backdrop-filter: blur(5px);
        box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
        animation: fadeIn 0.5s ease-out;
        justify-self: center;
        justify-content: center;
    }

    @keyframes fadeIn {
        from {
            opacity: 0;
            transform: translateY(-10px);
        }
        to {
            opacity: 1;
            transform: translateY(0);
        }
    }

    .voice-hint-content {
        display: flex;
        align-items: center;
        text-align: center;
        gap: 10px;
    }

    .voice-hint-content svg {
        flex-shrink: 0;
    }

    .error-message {
        color: #fff;
        text-align: center;
        margin: 20px auto;
        padding: 12px 15px;
        background-color: rgba(220, 53, 69, 0.8);
        border-radius: 8px;
        max-width: 600px;
        backdrop-filter: blur(10px);
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    }

    .loading-container {
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        margin: 50px 0;
        gap: 20px;
    }

    .loading-spinner {
        width: 50px;
        height: 50px;
        border: 5px solid rgba(255, 255, 255, 0.3);
        border-radius: 50%;
        border-top-color: white;
        animation: spin 1s ease-in-out infinite;
    }

    @keyframes spin {
        to {
            transform: rotate(360deg);
        }
    }

    .loading-text {
        font-size: 1.1rem;
        color: white;
    }

    .no-data {
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        margin: 50px auto;
        color: white;
        text-align: center;
        gap: 20px;
    }

    .no-data svg {
        opacity: 0.8;
    }

    .no-data p {
        font-size: 1.2rem;
        max-width: 400px;
    }


    .current-temp-icon img {
        width: 100px;
        height: 100px;
        filter: drop-shadow(0 0 8px rgba(255, 255, 255, 0.4));
    }

    .current-temp {
        display: flex;
        align-items: center;
        gap: 15px;
        margin-bottom: 10px;
    }


    @media (min-width: 1024px) {
        .weather-app {
            justify-content: center;
        }

        .next-weather {
            text-align: center;
        }

        .weather-card {
            text-align: center;
        }

        .today-weather-card {
            text-align: center;

        }
    }

    /* Für kleine Bildschirme */
    @media (max-width: 1023px) {
        .next-weather {
            justify-content: center; /* Karten zentrieren */
        }

        .app-header {
            text-align: center;
        }

        .next-weather {
            text-align: center;
        }

        .weather-card {
            text-align: center;
        }

        .today-weather-card {
            text-align: center;

        }

    }

    @media (max-width: 768px) {
        .app-header {
            align-items: stretch;
            text-align: center;
        }


        .current-temp {
            justify-content: center;
            text-align: center;

        }


        .city-input {
            width: 100%;
            padding: 12px 15px;
            font-size: 2rem;
            border: none;
            border-radius: 25px;
            background-color: rgba(255, 255, 255, 0.2);
            color: white;
            backdrop-filter: blur(10px);
            transition: all 0.3s ease;
        }

        .search-form {
            width: 100%;
        }

        .input-wrapper {
            flex: 1;
        }

        .next-weather {
            text-align: center;
        }

        .weather-card {
            text-align: center;
        }

        .today-weather-card {
            text-align: center;

        }

    }

    @media (max-width: 480px) {
        .weather-app {
            padding: 15px;
            text-align: center;

        }

        h1 {
            font-size: 1.5rem;
        }

        h2 {
            font-size: 1.3rem;
        }

        .next-weather {
            text-align: center;
        }

        .weather-card {
            text-align: center;
        }

        .today-weather-card {
            text-align: center;

        }

    }

    .weather-card:hover {
        transform: translateY(-5px);
    }


</style>
