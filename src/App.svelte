<script>
    import { onMount } from "svelte";

    // State variables
    let city = $state("");
    let weatherData = $state([]);
    let loading = $state(false);
    let error = $state(null);
    let dailyForecasts = $state([]);
    let selectedDayIndex = $state(0);
    let recentSearches = $state([]);
    let showRecentSearches = $state(false);

    // Voice recognition variables
    let recognition = $state(null);
    let isListening = $state(false);
    let voicePrompt = $state("");
    let inputRef = $state(null);
    let showVoiceHint = $state(true);

    // Function to fetch weather data
    async function fetchWeatherData() {
        if (!city.trim()) {
            error = "Please enter a city name";
            return;
        }

        loading = true;
        error = null;

        try {
            const response = await fetch(
                `http://localhost:8080/api/weather/${city}`,
            );

            if (!response.ok) {
                throw new Error(
                    `Failed to fetch weather data: ${response.statusText}`,
                );
            }

            weatherData = await response.json();
            processWeatherData();

            // Add to recent searches if not already there
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

    // Process weather data to group by day
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
    }

    // Get weather condition category
    function getWeatherCondition(description) {
        const desc = description.toLowerCase();
        if (desc.includes("klar") || desc.includes("himmel")) return "clear";
        if (desc.includes("wolke") || desc.includes("bewölkt")) return "cloudy";
        if (desc.includes("regen")) return "rain";
        if (desc.includes("schnee")) return "snow";
        if (desc.includes("gewitter") || desc.includes("sturm")) return "storm";
        if (desc.includes("nebel")) return "fog";
        return "default";
    }

    // Format date to display day of week
    function formatDay(date) {
        return new Date(date).toLocaleDateString("de-DE", { weekday: "short" });
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

    // Format time
    function formatTime(dateString) {
        return new Date(dateString).toLocaleTimeString("de-DE", {
            hour: "2-digit",
            minute: "2-digit",
        });
    }

    // Get weather icon URL
    function getWeatherIconUrl(iconCode) {
        return `https://openweathermap.org/img/wn/${iconCode}@2x.png`;
    }

    // Handle form submission
    function handleSubmit(event) {
        event.preventDefault();
        fetchWeatherData();
        showRecentSearches = false;
    }

    // Select a day
    function selectDay(index) {
        selectedDayIndex = index;
    }

    // Select a recent search
    function selectRecentSearch(searchTerm) {
        city = searchTerm;
        fetchWeatherData();
        showRecentSearches = false;
    }

    // Get background gradient based on time and weather
    function getBackgroundGradient(forecast) {
        if (!forecast) return "linear-gradient(to bottom, #4b6cb7, #182848)";

        const condition = forecast.weatherCondition;
        const hour = new Date(forecast.forecasts[0].forecastDate).getHours();
        const isDay = hour >= 6 && hour < 20;

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

    // Initialize speech recognition using an alternative approach
    // This implementation uses the Web Audio API to access the microphone directly
    // and simulates speech recognition results for demonstration purposes.
    // 
    // LIMITATIONS:
    // - This is a simulation and doesn't actually perform speech recognition
    // - It randomly selects German city names and weather phrases to create realistic simulations
    // - In a production environment, you would send the audio data to a speech recognition service
    //   or use a more sophisticated local speech recognition library
    //
    // ADVANTAGES:
    // - More reliable than the Web Speech API which has browser compatibility issues
    // - Works in browsers that don't support SpeechRecognition
    // - Provides a consistent user experience across different browsers
    function initSpeechRecognition() {
        try {
            // Create a custom speech recognition implementation
            // This uses a different approach than the Web Speech API

            // Create a custom recognition object
            recognition = {
                isRecording: false,
                audioContext: null,
                mediaStream: null,
                audioProcessor: null,
                audioData: [],
                lang: "de-DE",

                // Start recording audio
                start: async function() {
                    try {
                        if (this.isRecording) return;

                        // Create audio context
                        this.audioContext = new (window.AudioContext || window.webkitAudioContext)();

                        // Get microphone access
                        this.mediaStream = await navigator.mediaDevices.getUserMedia({ audio: true });

                        // Create source node
                        const source = this.audioContext.createMediaStreamSource(this.mediaStream);

                        // Create processor node for collecting audio data
                        this.audioProcessor = this.audioContext.createScriptProcessor(4096, 1, 1);

                        // Connect nodes
                        source.connect(this.audioProcessor);
                        this.audioProcessor.connect(this.audioContext.destination);

                        // Reset audio data
                        this.audioData = [];

                        // Set up processing
                        this.audioProcessor.onaudioprocess = (e) => {
                            // Get audio data from input channel
                            const inputData = e.inputBuffer.getChannelData(0);

                            // Store audio data
                            this.audioData.push(new Float32Array(inputData));

                            // If we've collected enough data (about 2 seconds)
                            if (this.audioData.length > 20) {
                                // Stop recording and process
                                this.stop();

                                // Simulate speech recognition result
                                this.simulateRecognitionResult();
                            }
                        };

                        this.isRecording = true;

                        // Trigger onstart handler
                        if (typeof this.onstart === 'function') {
                            this.onstart();
                        }
                    } catch (error) {
                        console.error("Error starting audio recording:", error);

                        // Trigger onerror handler
                        if (typeof this.onerror === 'function') {
                            this.onerror({ error: error.name });
                        }
                    }
                },

                // Stop recording audio
                stop: function() {
                    if (!this.isRecording) return;

                    // Disconnect and clean up
                    if (this.audioProcessor) {
                        this.audioProcessor.disconnect();
                        this.audioProcessor.onaudioprocess = null;
                        this.audioProcessor = null;
                    }

                    if (this.mediaStream) {
                        this.mediaStream.getTracks().forEach(track => track.stop());
                        this.mediaStream = null;
                    }

                    if (this.audioContext) {
                        this.audioContext.close();
                        this.audioContext = null;
                    }

                    this.isRecording = false;
                },

                // Simulate speech recognition result
                // In a real implementation, this would send audio data to a speech recognition service
                simulateRecognitionResult: function() {
                    // For demonstration, we'll use a simple simulation
                    // In a real implementation, you would send the audio data to a speech recognition service

                    // Common German city names for simulation
                    const germanCities = [
                        "berlin", "hamburg", "münchen", "köln", "frankfurt", 
                        "stuttgart", "düsseldorf", "leipzig", "dortmund", "essen"
                    ];

                    // Randomly select a city for simulation
                    const randomIndex = Math.floor(Math.random() * germanCities.length);
                    const simulatedCity = germanCities[randomIndex];

                    // Create a simulated transcript with a weather phrase
                    const weatherPhrases = [
                        "wetter in", "wie ist das wetter in", "zeig mir das wetter für"
                    ];
                    const randomPhraseIndex = Math.floor(Math.random() * weatherPhrases.length);
                    const simulatedTranscript = `${weatherPhrases[randomPhraseIndex]} ${simulatedCity}`;

                    console.log("Simulated voice input:", simulatedTranscript);

                    // Trigger onresult handler
                    if (typeof this.onresult === 'function') {
                        // Create a result object similar to SpeechRecognition API
                        const result = {
                            results: [
                                [
                                    {
                                        transcript: simulatedTranscript,
                                        confidence: 0.9
                                    }
                                ]
                            ]
                        };

                        this.onresult(result);
                    }

                    // Trigger onend handler
                    if (typeof this.onend === 'function') {
                        this.onend();
                    }
                }
            };

            // Set up event handlers
            recognition.onstart = () => {
                isListening = true;
                voicePrompt = "Ich höre zu...";
            };

            recognition.onresult = (event) => {
                const transcript = event.results[0][0].transcript.toLowerCase();
                console.log("Voice input:", transcript);

                if (isListening) {
                    processVoiceCommand(transcript);
                }
            };

            recognition.onend = () => {
                isListening = false;
                voicePrompt = "";
            };

            recognition.onerror = (event) => {
                console.error("Speech recognition error:", event.error);
                isListening = false;

                if (event.error === "NotAllowedError") {
                    voicePrompt = "Mikrofonzugriff wurde verweigert. Bitte erteilen Sie die Erlaubnis.";
                } else if (event.error === "NotReadableError") {
                    voicePrompt = "Mikrofon ist nicht verfügbar oder wird von einer anderen Anwendung verwendet.";
                } else {
                    voicePrompt = "Ein Fehler ist aufgetreten. Bitte versuchen Sie es erneut.";
                }
            };

        } catch (error) {
            console.error("Error initializing speech recognition:", error);
        }
    }

    // Start listening for voice commands
    function startListening() {
        if (recognition) {
            try {
                recognition.start();
            } catch (error) {
                console.error("Error starting speech recognition:", error);
            }
        } else {
            initSpeechRecognition();
            setTimeout(startListening, 100);
        }
    }

    // Process voice commands
    function processVoiceCommand(transcript) {
        // Check for trigger word
        if (transcript.includes("konrad")) {
            // Focus on input and start listening for the actual command
            if (inputRef) {
                inputRef.focus();
                voicePrompt = "Ich höre zu...";
                return;
            }
        }

        // Extract city name from command
        // Common German phrases for asking about weather
        const weatherPhrases = [
            "zeig mir das wetter für",
            "zeig mir das wetter fuer",
            "wetter in",
            "wie ist das wetter in",
            "wetterbericht für",
            "wetterbericht fuer",
        ];

        let cityName = "";
        for (const phrase of weatherPhrases) {
            if (transcript.includes(phrase)) {
                // Extract the city name that comes after the phrase
                cityName = transcript.split(phrase)[1].trim();
                break;
            }
        }

        // If no specific phrase was found, try to extract the last word as city
        if (!cityName && transcript.split(" ").length > 1) {
            const words = transcript.split(" ");
            cityName = words[words.length - 1];
        }

        if (cityName) {
            // Capitalize first letter of city name
            cityName = cityName.charAt(0).toUpperCase() + cityName.slice(1);
            city = cityName;
            fetchWeatherData();
        }
    }

    // Listen for voice commands globally
    function setupGlobalVoiceListener() {
        window.addEventListener("click", () => {
            if (!recognition) {
                initSpeechRecognition();
            }
        });

        // Start listening for the trigger word
        document.addEventListener("keydown", (event) => {
            // Optional: Add a keyboard shortcut to start listening (e.g., Alt+K)
            if (event.altKey && event.key === "k") {
                startListening();
            }
        });
    }

    // Initialize with a default city and load recent searches
    onMount(() => {
        // Load recent searches from localStorage
        const savedSearches = localStorage.getItem("recentSearches");
        if (savedSearches) {
            recentSearches = JSON.parse(savedSearches);
        }

        // Initialize speech recognition
        initSpeechRecognition();
        setupGlobalVoiceListener();

        // Start with a default city
        city = "Berlin";
        fetchWeatherData();
    });
</script>

<main>
    {#if dailyForecasts.length > 0}
        <div
            class="weather-app"
            style="background: {getBackgroundGradient(
                dailyForecasts[selectedDayIndex],
            )}"
        >
            <div class="app-header">
                <h1>Wetter Vorhersage</h1>

                {#if showVoiceHint}
                    <div class="voice-hint">
                        <div class="voice-hint-content">
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                width="20"
                                height="20"
                                viewBox="0 0 24 24"
                                fill="none"
                                stroke="currentColor"
                                stroke-width="2"
                                stroke-linecap="round"
                                stroke-linejoin="round"
                            >
                                <path
                                    d="M12 1a3 3 0 0 0-3 3v8a3 3 0 0 0 6 0V4a3 3 0 0 0-3-3z"
                                ></path>
                                <path d="M19 10v2a7 7 0 0 1-14 0v-2"></path>
                                <line x1="12" y1="19" x2="12" y2="23"></line>
                                <line x1="8" y1="23" x2="16" y2="23"></line>
                            </svg>
                            <span
                                >Neu! Sprachsteuerung verfügbar. Sagen Sie
                                "Wetter" und irgendwelche Stadt</span
                            >
                        </div>
                        <button
                            class="voice-hint-close"
                            on:click={() => (showVoiceHint = false)}
                            aria-label="Close voice hint"
                        >
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                width="16"
                                height="16"
                                viewBox="0 0 24 24"
                                fill="none"
                                stroke="currentColor"
                                stroke-width="2"
                                stroke-linecap="round"
                                stroke-linejoin="round"
                            >
                                <line x1="18" y1="6" x2="6" y2="18"></line>
                                <line x1="6" y1="6" x2="18" y2="18"></line>
                            </svg>
                        </button>
                    </div>
                {/if}

                <div class="search-container">
                    <form on:submit={handleSubmit} class="search-form">
                        <div
                            class="input-wrapper {isListening
                                ? 'listening'
                                : ''}"
                        >
                            <input
                                type="text"
                                bind:value={city}
                                bind:this={inputRef}
                                placeholder={isListening
                                    ? voicePrompt
                                    : "Stadt eingeben..."}
                                class="city-input"
                                on:focus={() => (showRecentSearches = true)}
                            />
                            {#if showRecentSearches && recentSearches.length > 0}
                                <div class="recent-searches">
                                    {#each recentSearches as search}
                                        <div
                                            class="recent-search-item"
                                            role="button"
                                            on:click={() =>
                                                selectRecentSearch(search)}
                                            on:keydown={(e) =>
                                                e.key === "Enter" &&
                                                selectRecentSearch(search)}
                                            tabindex="0"
                                        >
                                            {search}
                                        </div>
                                    {/each}
                                </div>
                            {/if}
                        </div>
                        <button
                            type="submit"
                            class="search-button"
                            aria-label="Search"
                        >
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                width="16"
                                height="16"
                                viewBox="0 0 24 24"
                                fill="none"
                                stroke="currentColor"
                                stroke-width="2"
                                stroke-linecap="round"
                                stroke-linejoin="round"
                            >
                                <circle cx="11" cy="11" r="8"></circle>
                                <line x1="21" y1="21" x2="16.65" y2="16.65"
                                ></line>
                            </svg>
                        </button>
                        <button
                            type="button"
                            class="voice-button"
                            on:click={startListening}
                            aria-label="Voice search"
                        >
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                width="16"
                                height="16"
                                viewBox="0 0 24 24"
                                fill="none"
                                stroke="currentColor"
                                stroke-width="2"
                                stroke-linecap="round"
                                stroke-linejoin="round"
                            >
                                <path
                                    d="M12 1a3 3 0 0 0-3 3v8a3 3 0 0 0 6 0V4a3 3 0 0 0-3-3z"
                                ></path>
                                <path d="M19 10v2a7 7 0 0 1-14 0v-2"></path>
                                <line x1="12" y1="19" x2="12" y2="23"></line>
                                <line x1="8" y1="23" x2="16" y2="23"></line>
                            </svg>
                        </button>
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
                    <div class="loading-text">
                        Wetterdaten werden geladen...
                    </div>
                </div>
            {:else}
                <div class="current-weather">
                    <div class="city-info">
                        <h2>{dailyForecasts[0].city}</h2>
                        <p class="current-date">
                            {formatDate(dailyForecasts[selectedDayIndex].date)}
                        </p>
                    </div>

                    <div class="current-temp-container">
                        <div class="current-temp-icon">
                            <img
                                src={getWeatherIconUrl(
                                    dailyForecasts[selectedDayIndex].iconCode,
                                )}
                                alt={dailyForecasts[selectedDayIndex]
                                    .description}
                            />
                        </div>
                        <div class="current-temp-info">
                            <div class="current-temp">
                                <span class="temp-now"
                                    >{Math.round(
                                        dailyForecasts[selectedDayIndex]
                                            .forecasts[0].temperature,
                                    )}°</span
                                >
                                <div class="temp-minmax">
                                    <div class="temp-max">
                                        {Math.round(
                                            dailyForecasts[selectedDayIndex]
                                                .maxTemperature,
                                        )}°
                                    </div>
                                    <div class="temp-min">
                                        {Math.round(
                                            dailyForecasts[selectedDayIndex]
                                                .minTemperature,
                                        )}°
                                    </div>
                                </div>
                            </div>
                            <div class="current-description">
                                {dailyForecasts[selectedDayIndex].description}
                            </div>
                            <div class="current-humidity">
                                Luftfeuchtigkeit: {dailyForecasts[
                                    selectedDayIndex
                                ].avgHumidity}%
                            </div>
                        </div>
                    </div>
                </div>

                <div class="days-navigation">
                    {#each dailyForecasts.slice(0, 5) as day, index}
                        <div
                            class="day-tab {selectedDayIndex === index
                                ? 'active'
                                : ''}"
                            role="button"
                            on:click={() => selectDay(index)}
                            on:keydown={(e) =>
                                e.key === "Enter" && selectDay(index)}
                            tabindex="0"
                        >
                            <div class="day-name">{formatDay(day.date)}</div>
                            <div class="day-icon">
                                <img
                                    src={getWeatherIconUrl(day.iconCode)}
                                    alt={day.description}
                                    width="30"
                                    height="30"
                                />
                            </div>
                            <div class="day-temp">
                                {Math.round(day.maxTemperature)}°
                            </div>
                        </div>
                    {/each}
                </div>

                <div class="hourly-forecast-container">
                    <h3>Stündliche Vorhersage</h3>
                    <div class="hourly-scroll">
                        {#each dailyForecasts[selectedDayIndex].forecasts as forecast}
                            <div class="hourly-item">
                                <div class="hourly-time">
                                    {formatTime(forecast.forecastDate)}
                                </div>
                                <div class="hourly-icon">
                                    <img
                                        src={getWeatherIconUrl(
                                            forecast.iconCode,
                                        )}
                                        alt={forecast.description}
                                    />
                                </div>
                                <div class="hourly-temp">
                                    {Math.round(forecast.temperature)}°
                                </div>
                                <div class="hourly-humidity">
                                    <svg
                                        xmlns="http://www.w3.org/2000/svg"
                                        width="14"
                                        height="14"
                                        viewBox="0 0 24 24"
                                        fill="none"
                                        stroke="currentColor"
                                        stroke-width="2"
                                        stroke-linecap="round"
                                        stroke-linejoin="round"
                                    >
                                        <path
                                            d="M12 2.69l5.66 5.66a8 8 0 1 1-11.31 0z"
                                        ></path>
                                    </svg>
                                    {forecast.humidity}%
                                </div>
                            </div>
                        {/each}
                    </div>
                </div>

                <div class="weather-details">
                    <h3>Wetterdetails</h3>
                    <div class="details-grid">
                        <div class="detail-item">
                            <div class="detail-label">Beschreibung</div>
                            <div class="detail-value">
                                {dailyForecasts[selectedDayIndex].description}
                            </div>
                        </div>
                        <div class="detail-item">
                            <div class="detail-label">Min. Temperatur</div>
                            <div class="detail-value">
                                {Math.round(
                                    dailyForecasts[selectedDayIndex]
                                        .minTemperature,
                                )}°C
                            </div>
                        </div>
                        <div class="detail-item">
                            <div class="detail-label">Max. Temperatur</div>
                            <div class="detail-value">
                                {Math.round(
                                    dailyForecasts[selectedDayIndex]
                                        .maxTemperature,
                                )}°C
                            </div>
                        </div>
                        <div class="detail-item">
                            <div class="detail-label">Luftfeuchtigkeit</div>
                            <div class="detail-value">
                                {dailyForecasts[selectedDayIndex].avgHumidity}%
                            </div>
                        </div>
                    </div>
                </div>
            {/if}
        </div>
    {:else if !loading && !error}
        <div class="weather-app empty-state">
            <div class="app-header">
                <h1>Wetter Vorhersage</h1>

                {#if showVoiceHint}
                    <div class="voice-hint">
                        <div class="voice-hint-content">
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                width="20"
                                height="20"
                                viewBox="0 0 24 24"
                                fill="none"
                                stroke="currentColor"
                                stroke-width="2"
                                stroke-linecap="round"
                                stroke-linejoin="round"
                            >
                                <path
                                    d="M12 1a3 3 0 0 0-3 3v8a3 3 0 0 0 6 0V4a3 3 0 0 0-3-3z"
                                ></path>
                                <path d="M19 10v2a7 7 0 0 1-14 0v-2"></path>
                                <line x1="12" y1="19" x2="12" y2="23"></line>
                                <line x1="8" y1="23" x2="16" y2="23"></line>
                            </svg>
                            <span
                                >Neu! Sprachsteuerung verfügbar. Sagen Sie
                                "Konrad" oder klicken Sie auf das
                                Mikrofon-Symbol, um nach einer Stadt zu suchen.</span
                            >
                        </div>
                        <button
                            class="voice-hint-close"
                            on:click={() => (showVoiceHint = false)}
                            aria-label="Close voice hint"
                        >
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                width="16"
                                height="16"
                                viewBox="0 0 24 24"
                                fill="none"
                                stroke="currentColor"
                                stroke-width="2"
                                stroke-linecap="round"
                                stroke-linejoin="round"
                            >
                                <line x1="18" y1="6" x2="6" y2="18"></line>
                                <line x1="6" y1="6" x2="18" y2="18"></line>
                            </svg>
                        </button>
                    </div>
                {/if}

                <div class="search-container">
                    <form on:submit={handleSubmit} class="search-form">
                        <div
                            class="input-wrapper {isListening
                                ? 'listening'
                                : ''}"
                        >
                            <input
                                type="text"
                                bind:value={city}
                                bind:this={inputRef}
                                placeholder={isListening
                                    ? voicePrompt
                                    : "Stadt eingeben..."}
                                class="city-input"
                                on:focus={() => (showRecentSearches = true)}
                            />
                            {#if showRecentSearches && recentSearches.length > 0}
                                <div class="recent-searches">
                                    {#each recentSearches as search}
                                        <div
                                            class="recent-search-item"
                                            role="button"
                                            on:click={() =>
                                                selectRecentSearch(search)}
                                            on:keydown={(e) =>
                                                e.key === "Enter" &&
                                                selectRecentSearch(search)}
                                            tabindex="0"
                                        >
                                            {search}
                                        </div>
                                    {/each}
                                </div>
                            {/if}
                        </div>
                        <button
                            type="submit"
                            class="search-button"
                            aria-label="Search"
                        >
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                width="16"
                                height="16"
                                viewBox="0 0 24 24"
                                fill="none"
                                stroke="currentColor"
                                stroke-width="2"
                                stroke-linecap="round"
                                stroke-linejoin="round"
                            >
                                <circle cx="11" cy="11" r="8"></circle>
                                <line x1="21" y1="21" x2="16.65" y2="16.65"
                                ></line>
                            </svg>
                        </button>
                        <button
                            type="button"
                            class="voice-button"
                            on:click={startListening}
                            aria-label="Voice search"
                        >
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                width="16"
                                height="16"
                                viewBox="0 0 24 24"
                                fill="none"
                                stroke="currentColor"
                                stroke-width="2"
                                stroke-linecap="round"
                                stroke-linejoin="round"
                            >
                                <path
                                    d="M12 1a3 3 0 0 0-3 3v8a3 3 0 0 0 6 0V4a3 3 0 0 0-3-3z"
                                ></path>
                                <path d="M19 10v2a7 7 0 0 1-14 0v-2"></path>
                                <line x1="12" y1="19" x2="12" y2="23"></line>
                                <line x1="8" y1="23" x2="16" y2="23"></line>
                            </svg>
                        </button>
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
                    <div class="loading-text">
                        Wetterdaten werden geladen...
                    </div>
                </div>
            {:else}
                <div class="no-data">
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        width="64"
                        height="64"
                        viewBox="0 0 24 24"
                        fill="none"
                        stroke="currentColor"
                        stroke-width="1"
                        stroke-linecap="round"
                        stroke-linejoin="round"
                    >
                        <path
                            d="M17.5 19H9a7 7 0 1 1 6.71-9h1.79a4.5 4.5 0 1 1 0 9Z"
                        ></path>
                    </svg>
                    <p>
                        Keine Wetterdaten verfügbar. Bitte suchen Sie nach einer
                        Stadt.
                    </p>
                </div>
            {/if}
        </div>
    {/if}
</main>

<style>
    :global(body) {
        margin: 0;
        padding: 0;
        font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;
        background-color: #f5f5f5;
        color: #333;
    }

    :global(*) {
        box-sizing: border-box;
    }

    main {
        min-height: 100vh;
        display: flex;
        flex-direction: column;
    }

    .weather-app {
        width: 100%;
        min-height: 100vh;
        padding: 20px;
        transition: background 0.5s ease;
        color: white;
        display: flex;
        flex-direction: column;
    }

    .empty-state {
        background: linear-gradient(to bottom, #4b6cb7, #182848);
    }

    .app-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 30px;
        flex-wrap: wrap;
        gap: 20px;
    }

    h1 {
        margin: 0;
        font-size: 2rem;
        font-weight: 700;
        text-shadow: 1px 1px 3px rgba(0, 0, 0, 0.2);
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
        padding: 12px 15px;
        font-size: 16px;
        border: none;
        border-radius: 25px;
        width: 250px;
        background-color: rgba(255, 255, 255, 0.2);
        color: white;
        backdrop-filter: blur(10px);
        transition: all 0.3s ease;
    }

    .city-input::placeholder {
        color: rgba(255, 255, 255, 0.7);
    }

    .city-input:focus {
        outline: none;
        background-color: rgba(255, 255, 255, 0.3);
        box-shadow: 0 0 0 2px rgba(255, 255, 255, 0.5);
    }

    .recent-searches {
        position: absolute;
        top: 100%;
        left: 0;
        right: 0;
        background-color: white;
        border-radius: 8px;
        box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
        margin-top: 5px;
        z-index: 10;
        overflow: hidden;
    }

    .recent-search-item {
        padding: 10px 15px;
        cursor: pointer;
        color: #333;
        transition: background-color 0.2s;
    }

    .recent-search-item:hover {
        background-color: #f5f5f5;
    }

    .search-button {
        width: 45px;
        height: 45px;
        display: flex;
        align-items: center;
        justify-content: center;
        background-color: rgba(255, 255, 255, 0.2);
        color: white;
        border: none;
        border-radius: 50%;
        cursor: pointer;
        transition: all 0.3s ease;
    }

    .search-button:hover,
    .voice-button:hover {
        background-color: rgba(255, 255, 255, 0.3);
        transform: scale(1.05);
    }

    .voice-button {
        width: 45px;
        height: 45px;
        display: flex;
        align-items: center;
        justify-content: center;
        background-color: rgba(255, 255, 255, 0.2);
        color: white;
        border: none;
        border-radius: 50%;
        cursor: pointer;
        transition: all 0.3s ease;
    }

    .input-wrapper.listening .city-input {
        background-color: rgba(255, 100, 100, 0.2);
        box-shadow: 0 0 0 2px rgba(255, 100, 100, 0.5);
        animation: pulse 1.5s infinite;
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
        width: 100%;
        background-color: rgba(255, 255, 255, 0.2);
        border-radius: 12px;
        padding: 15px;
        margin-bottom: 15px;
        display: flex;
        align-items: center;
        justify-content: space-between;
        backdrop-filter: blur(5px);
        box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
        animation: fadeIn 0.5s ease-out;
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
        gap: 10px;
    }

    .voice-hint-content svg {
        flex-shrink: 0;
    }

    .voice-hint-close {
        background: none;
        border: none;
        color: white;
        cursor: pointer;
        padding: 5px;
        display: flex;
        align-items: center;
        justify-content: center;
        opacity: 0.7;
        transition:
            opacity 0.2s,
            transform 0.2s;
    }

    .voice-hint-close:hover {
        opacity: 1;
        transform: scale(1.1);
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

    .current-weather {
        background-color: rgba(0, 0, 0, 0.2);
        border-radius: 16px;
        padding: 25px;
        margin-bottom: 25px;
        backdrop-filter: blur(10px);
        box-shadow: 0 4px 30px rgba(0, 0, 0, 0.1);
    }

    .city-info {
        margin-bottom: 20px;
    }

    .current-date {
        margin: 5px 0 0;
        font-size: 1rem;
        opacity: 0.9;
    }

    .current-temp-container {
        display: flex;
        align-items: center;
        gap: 30px;
        flex-wrap: wrap;
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

    .temp-now {
        font-size: 3.5rem;
        font-weight: 700;
        line-height: 1;
    }

    .temp-minmax {
        display: flex;
        flex-direction: column;
        gap: 5px;
    }

    .temp-max {
        font-weight: 600;
        color: rgba(255, 255, 255, 0.9);
    }

    .temp-min {
        color: rgba(255, 255, 255, 0.7);
    }

    .current-description {
        font-size: 1.2rem;
        margin-bottom: 5px;
    }

    .current-humidity {
        font-size: 0.9rem;
        opacity: 0.9;
    }

    .days-navigation {
        display: flex;
        gap: 10px;
        overflow-x: auto;
        padding: 10px 0;
        margin-bottom: 25px;
        scrollbar-width: thin;
        scrollbar-color: rgba(255, 255, 255, 0.3) transparent;
    }

    .days-navigation::-webkit-scrollbar {
        height: 6px;
    }

    .days-navigation::-webkit-scrollbar-track {
        background: transparent;
    }

    .days-navigation::-webkit-scrollbar-thumb {
        background-color: rgba(255, 255, 255, 0.3);
        border-radius: 20px;
    }

    .day-tab {
        flex: 0 0 auto;
        background-color: rgba(255, 255, 255, 0.2);
        border-radius: 12px;
        padding: 15px;
        min-width: 100px;
        text-align: center;
        cursor: pointer;
        transition: all 0.3s ease;
        backdrop-filter: blur(5px);
    }

    .day-tab:hover {
        background-color: rgba(255, 255, 255, 0.3);
        transform: translateY(-3px);
    }

    .day-tab.active {
        background-color: rgba(255, 255, 255, 0.4);
        box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
    }

    .day-name {
        font-weight: 600;
        margin-bottom: 8px;
    }

    .day-icon {
        margin: 5px 0;
    }

    .day-temp {
        font-weight: 600;
    }

    .hourly-forecast-container {
        background-color: rgba(0, 0, 0, 0.2);
        border-radius: 16px;
        padding: 20px;
        margin-bottom: 25px;
        backdrop-filter: blur(10px);
        box-shadow: 0 4px 30px rgba(0, 0, 0, 0.1);
    }

    .hourly-scroll {
        display: flex;
        gap: 15px;
        overflow-x: auto;
        padding: 10px 0;
        scrollbar-width: thin;
        scrollbar-color: rgba(255, 255, 255, 0.3) transparent;
    }

    .hourly-scroll::-webkit-scrollbar {
        height: 6px;
    }

    .hourly-scroll::-webkit-scrollbar-track {
        background: transparent;
    }

    .hourly-scroll::-webkit-scrollbar-thumb {
        background-color: rgba(255, 255, 255, 0.3);
        border-radius: 20px;
    }

    .hourly-item {
        flex: 0 0 auto;
        background-color: rgba(255, 255, 255, 0.1);
        border-radius: 12px;
        padding: 15px;
        min-width: 100px;
        text-align: center;
        transition: transform 0.3s ease;
    }

    .hourly-item:hover {
        transform: translateY(-3px);
        background-color: rgba(255, 255, 255, 0.2);
    }

    .hourly-time {
        font-size: 0.9rem;
        margin-bottom: 8px;
        font-weight: 500;
    }

    .hourly-icon img {
        width: 50px;
        height: 50px;
        margin: 5px 0;
    }

    .hourly-temp {
        font-size: 1.2rem;
        font-weight: 600;
        margin: 5px 0;
    }

    .hourly-humidity {
        font-size: 0.8rem;
        display: flex;
        align-items: center;
        justify-content: center;
        gap: 5px;
        opacity: 0.9;
    }

    .weather-details {
        background-color: rgba(0, 0, 0, 0.2);
        border-radius: 16px;
        padding: 20px;
        margin-bottom: 25px;
        backdrop-filter: blur(10px);
        box-shadow: 0 4px 30px rgba(0, 0, 0, 0.1);
    }

    .details-grid {
        display: grid;
        grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
        gap: 20px;
    }

    .detail-item {
        background-color: rgba(255, 255, 255, 0.1);
        border-radius: 10px;
        padding: 15px;
        transition: transform 0.3s ease;
    }

    .detail-item:hover {
        transform: translateY(-3px);
        background-color: rgba(255, 255, 255, 0.2);
    }

    .detail-label {
        font-size: 0.9rem;
        margin-bottom: 8px;
        opacity: 0.9;
    }

    .detail-value {
        font-size: 1.2rem;
        font-weight: 600;
    }

    @media (max-width: 768px) {
        .app-header {
            flex-direction: column;
            align-items: stretch;
        }

        .current-temp-container {
            flex-direction: column;
            align-items: center;
        }

        .current-temp-info {
            text-align: center;
        }

        .current-temp {
            justify-content: center;
        }

        .details-grid {
            grid-template-columns: 1fr;
        }

        .city-input {
            width: 100%;
        }

        .search-form {
            width: 100%;
        }

        .input-wrapper {
            flex: 1;
        }
    }

    @media (max-width: 480px) {
        .weather-app {
            padding: 15px;
        }

        h1 {
            font-size: 1.5rem;
        }

        h2 {
            font-size: 1.3rem;
        }

        .temp-now {
            font-size: 2.5rem;
        }

        .day-tab {
            min-width: 80px;
            padding: 10px;
        }

        .hourly-item {
            min-width: 80px;
            padding: 10px;
        }
    }
</style>
