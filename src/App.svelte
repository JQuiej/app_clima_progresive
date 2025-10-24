<main>
    <h1>Clima Actual</h1>

    <div class="form-group">
        <div>
            <label for="latitude">Latitud</label>
            <input type="text" id="latitude" bind:value={lat}>
        </div>
        <div>
            <label for="longitude">Longitud</label>
            <input type="text" id="longitude" bind:value={lon}>
            <p>Coordenadas por defecto.</p>
        </div>
        <button on:click={fetchWeather}>
            Buscar Clima Manualmente
        </button>
    </div>

    {#if isLoading}
        <div id="loading">
            <div class="loader"></div>
        </div>
    {/if}

    {#if weatherData}
        <div id="weatherResult">
            
            <div id="weatherIcon">{getWeatherInfo(weatherData.current.weather_code).icon}</div>
            <div id="temperature">{Math.round(weatherData.current.temperature_2m)}°C</div>
            <div id="description">{getWeatherInfo(weatherData.current.weather_code).description}</div>
            <div id="location">Clima para: {weatherData.timezone}</div>

            <div class="current-details-grid">
                <div class="detail-box">
                    <span class="detail-label">Sensación</span>
                    <span class="detail-value">{Math.round(weatherData.current.apparent_temperature)}°C</span>
                </div>
                <div class="detail-box">
                    <span class="detail-label">Humedad</span>
                    <span class="detail-value">{weatherData.current.relative_humidity_2m}%</span>
                </div>
                <div class="detail-box">
                    <span class="detail-label">Viento</span>
                    <span class="detail-value">{weatherData.current.wind_speed_10m} km/h</span>
                </div>
                <div class="detail-box">
                    <span class="detail-label">Precipitación</span>
                    <span class="detail-value">{weatherData.current.precipitation} mm</span>
                </div>
            </div>

            <div id="hourlyForecastContainer">
                <h3>Pronóstico por Hora</h3>
                <div id="hourlyForecast">
                    {#each visibleHourlyForecast as hour}
                        <div class="hourly-item">
                            <span class="hourly-time">{hour.time}</span>
                            <span class="hourly-icon">{hour.icon}</span>
                            <span class="hourly-temp">{hour.temp}°</span>
                        </div>
                    {/each}
                </div>
            </div>

            <div id="forecast">
                <h3>Pronóstico 7 Días</h3>
                <div id="forecastDays">
                    {#each weatherData.daily.time as time, index}
                        <div class="forecast-day">
                            <div class="day-name">{new Date(time).toLocaleDateString('es-ES', { weekday: 'short' })}</div>
                            <div class="day-icon">{getWeatherInfo(weatherData.daily.weather_code[index]).icon}</div>
                            <div class="day-temp">
                                <span>{Math.round(weatherData.daily.temperature_2m_max[index])}°</span> / 
                                <span class="temp-min">{Math.round(weatherData.daily.temperature_2m_min[index])}°</span>
                            </div>
                        </div>
                    {/each}
                </div>
            </div>

        </div>
    {/if}
</main>


<script>
    import { onMount } from 'svelte'; // onMount es como "window.addEventListener('load')"

    // --- Variables de Estado (Estado Reactivo) ---
    // Svelte actualiza el HTML automáticamente cuando estas variables cambian
    let lat = "19.4326";
    let lon = "-99.1332";
    let isLoading = false;
    let weatherData = null; // Guardará todos los datos de la API
    let visibleHourlyForecast = []; // Un array para el pronóstico por hora

    // --- Lógica de la API ---

    // onMount se ejecuta una vez que el componente está en la página
    onMount(() => {
        // Mantenemos tu lógica de geolocalización
        if ('geolocation' in navigator) {
            navigator.geolocation.getCurrentPosition(
                (position) => {
                    lat = position.coords.latitude.toFixed(4);
                    lon = position.coords.longitude.toFixed(4);
                    fetchWeather(); // Carga el clima con las nuevas coordenadas
                },
                (error) => {
                    console.warn(`Error al obtener ubicación (${error.code}): ${error.message}`);
                    alert('No se pudo obtener tu ubicación. Usando valores por defecto.');
                    fetchWeather(); // Carga el clima con valores por defecto
                }
            );
        } else {
            alert('Geolocalización no soportada. Usando valores por defecto.');
            fetchWeather(); // Carga el clima con valores por defecto
        }
    });
    
    // Función para obtener el clima (casi idéntica)
    async function fetchWeather() {
        if (!lat || !lon) {
            alert('Por favor, ingresa latitud y longitud.');
            return;
        }

        isLoading = true; // Svelte mostrará el loader
        weatherData = null; // Oculta resultados antiguos

        const apiUrl = `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&current=temperature_2m,weather_code,apparent_temperature,relative_humidity_2m,precipitation,wind_speed_10m&hourly=temperature_2m,weather_code&daily=weather_code,temperature_2m_max,temperature_2m_min&timezone=auto&forecast_days=7`;

        try {
            const response = await fetch(apiUrl);
            if (!response.ok) throw new Error('No se pudo obtener la respuesta de la API');
            
            const data = await response.json();
            weatherData = data; // ¡Magia! Svelte actualiza el HTML
            
            // Procesamos el pronóstico por hora
            processHourlyForecast(data.hourly);

        } catch (error) {
            console.error('Error al obtener el clima:', error);
            alert('Error al obtener el clima. Inténtalo de nuevo.');
        } finally {
            isLoading = false; // Svelte oculta el loader
        }
    }

    // Procesador para el pronóstico por hora
    function processHourlyForecast(hourly) {
        const now = new Date();
        const currentHourIndex = hourly.time.findIndex(time => new Date(time) >= now) || 0;
        const next24Hours = hourly.time.slice(currentHourIndex, currentHourIndex + 24);
        
        // Svelte necesita que actualicemos la variable del 'each'
        visibleHourlyForecast = next24Hours.map((time, index) => {
            const realIndex = currentHourIndex + index;
            return {
                time: new Date(time).getHours().toString().padStart(2, '0') + ":00",
                icon: getWeatherInfo(hourly.weather_code[realIndex]).icon,
                temp: Math.round(hourly.temperature_2m[realIndex])
            };
        });
    }

    // Helper de códigos de clima (sin cambios)
    function getWeatherInfo(code) {
        switch (code) {
            case 0: return { description: "Despejado", icon: "☀️" };
            case 1:
            case 2:
            case 3: return { description: "Parcialmente Nublado", icon: "🌤️" };
            case 45:
            case 48: return { description: "Niebla", icon: "🌫️" };
            case 51:
            case 53:
            case 55: return { description: "Llovizna", icon: "🌦️" };
            case 61:
            case 63:
            case 65: return { description: "Lluvia", icon: "🌧️" };
            case 66:
            case 67: return { description: "Lluvia Helada", icon: "🌨️" };
            case 71:
            case 73:
            case 75: return { description: "Nieve", icon: "❄️" };
            case 77: return { description: "Granizo", icon: "🌨️" };
            case 80:
            case 81:
            case 82: return { description: "Chubascos", icon: "🌧️" };
            case 95:
            case 96:
            case 99: return { description: "Tormenta", icon: "⛈️" };
            default: return { description: "Desconocido", icon: "❓" };
        }
    }
</script>


<style>
    /* 1. Reset Básico y Tema Global */
    * {
        margin: 0;
        padding: 0;
        box-sizing: border-box;
    }

    /* Svelte aplica los estilos a :global() 
      para afectar 'body' y 'html' 
    */
    :global(html, body) {
        height: 100%;
    }

    :global(body) {
        font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
        background-color: #1a1a1a;
        color: #e0e0e0;
        display: flex;
        align-items: center;
        justify-content: center;
        padding: 1rem;
    }

    /* 2. Contenedor Principal */
    /* 'main' ahora está "scopado" (limitado) a este componente */
    main {
        width: 100%;
        max-width: 450px;
        background-color: #2a2a2a;
        border-radius: 12px;
        padding: 2rem;
        box-shadow: 0 10px 25px rgba(0, 0, 0, 0.3);
    }

    h1 {
        font-size: 2rem;
        font-weight: 700;
        text-align: center;
        margin-bottom: 1.5rem;
        color: #ffffff;
    }

    /* 3. Estilos del Formulario */
    .form-group > div {
        margin-bottom: 1rem;
    }

    label {
        display: block;
        font-size: 0.9rem;
        margin-bottom: 0.5rem;
        color: #aaaaaa;
    }

    input[type="text"] {
        width: 100%;
        padding: 0.75rem;
        background-color: #333333;
        border: 1px solid #444444;
        border-radius: 8px;
        color: #e0e0e0;
        font-size: 1rem;
        transition: border-color 0.3s, box-shadow 0.3s;
    }

    input[type="text"]:focus {
        outline: none;
        border-color: #0ea5e9;
        box-shadow: 0 0 0 3px rgba(14, 165, 233, 0.3);
    }

    p {
        font-size: 0.75rem;
        color: #888888;
        margin-top: 0.25rem;
    }

    button {
        width: 100%;
        background-color: #0ea5e9;
        color: #ffffff;
        font-weight: 700;
        padding: 0.75rem;
        border: none;
        border-radius: 8px;
        cursor: pointer;
        transition: background-color 0.3s;
        font-size: 1rem;
        margin-top: 1rem;
    }

    button:hover {
        background-color: #0c85b8;
    }

    /* 4. Loader */
    #loading {
        /* Ya no necesitamos 'display: none' */
        justify-content: center;
        align-items: center;
        height: 80px;
    }

    .loader {
        border: 4px solid #444444; 
        border-top: 4px solid #0ea5e9;
        border-radius: 50%;
        width: 40px;
        height: 40px;
        animation: spin 1s linear infinite;
    }

    @keyframes spin {
        0% { transform: rotate(0deg); }
        100% { transform: rotate(360deg); }
    }

    /* 5. Resultados del Clima */
    #weatherResult {
        /* Ya no necesitamos 'display: none' */
        text-align: center;
    }

    #weatherIcon {
        font-size: 5rem;
        margin-bottom: 1rem;
    }

    #temperature {
        font-size: 3.5rem;
        font-weight: 700;
        color: #ffffff;
    }

    #description {
        font-size: 1.5rem;
        color: #aaaaaa;
        text-transform: capitalize;
        margin-top: 0.5rem;
    }

    #location {
        font-size: 1rem;
        color: #888888;
        margin-top: 1rem;
    }

    /* 6. Grid de Detalles */
    .current-details-grid {
        display: grid;
        grid-template-columns: 1fr 1fr;
        gap: 1rem;
        margin-top: 2rem;
        text-align: left;
    }

    .detail-box {
        background-color: #333333;
        padding: 1rem;
        border-radius: 8px;
    }

    .detail-label {
        display: block;
        font-size: 0.8rem;
        color: #aaaaaa;
        margin-bottom: 0.25rem;
    }

    .detail-value {
        display: block;
        font-size: 1.1rem;
        font-weight: 600;
        color: #ffffff;
    }

    /* 7. Pronóstico por Hora */
    #hourlyForecastContainer {
        margin-top: 2rem;
        border-top: 1px solid #444444;
        padding-top: 1.5rem;
    }

    #hourlyForecastContainer h3 {
        font-size: 1.1rem;
        font-weight: 600;
        margin-bottom: 1rem;
        text-align: center;
    }

    #hourlyForecast {
        display: flex;
        gap: 0.75rem;
        overflow-x: auto; 
        padding: 1rem 0.25rem;
        -webkit-overflow-scrolling: touch; 
    }

    .hourly-item {
        flex: 0 0 70px;
        display: flex;
        flex-direction: column;
        align-items: center;
        background-color: #333333;
        padding: 0.75rem;
        border-radius: 8px;
    }

    .hourly-time {
        font-size: 0.8rem;
        color: #aaaaaa;
    }

    .hourly-icon {
        font-size: 1.75rem;
        margin: 0.5rem 0;
    }

    .hourly-temp {
        font-size: 1rem;
        font-weight: 600;
    }

    /* 8. Pronóstico Diario */
    #forecast {
        margin-top: 2rem;
        border-top: 1px solid #444444;
        padding-top: 1.5rem;
    }

    #forecast h3 {
        font-size: 1.1rem;
        font-weight: 600;
        margin-bottom: 1rem;
        text-align: center;
    }

    #forecastDays {
        display: flex;
        justify-content: space-between;
        gap: 0.5rem;
        text-align: center;
        overflow-x: auto; 
    }

    .forecast-day {
        flex: 1;
        min-width: 60px;
        background-color: #333333;
        padding: 1rem 0.5rem;
        border-radius: 8px;
    }

    .day-name {
        font-weight: 600;
        font-size: 0.9rem;
    }

    .day-icon {
        font-size: 2rem;
        margin: 0.5rem 0;
    }

    .day-temp {
        font-size: 0.9rem;
    }

    .day-temp .temp-min {
        color: #888888;
    }
</style>