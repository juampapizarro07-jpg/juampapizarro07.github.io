<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Control del horno</title>
    <style>
        /* ===== BLOQUE CSS: aspecto de la pagina (colores oscuros, tarjetas) ===== */
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background-color: #1e1e2e;
            color: #cdd6f4;
            display: flex;
            justify-content: center;
            align-items: flex-start;
            min-height: 100vh;
            margin: 0;
        }
        .wrap { width: 380px; max-width: 94vw; margin: 16px; }  /* Columna central, se adapta al celular */
        .card {                                                 /* Cada tarjeta (Monitoreo, Registro, Configurar) */
            background-color: #313244;
            padding: 22px;
            border-radius: 12px;
            box-shadow: 0 4px 15px rgba(0,0,0,0.5);
            text-align: center;
            margin-bottom: 16px;
        }
        h2 { margin: 0 0 12px; color: #89b4fa; }
        h3 { margin: 0 0 12px; color: #89b4fa; font-size: 1.05em; }

        /* --- Tarjeta superior: titulo a la izquierda, pilotos a la derecha --- */
        .head { display: flex; justify-content: space-between; align-items: center; gap: 10px; text-align: left; }
        .head h2 { margin: 0; }
        .pilotos { display: flex; gap: 12px; }                  /* Los tres pilotos en fila */
        .piloto { text-align: center; font-size: 0.7em; color: #a6adc8; min-width: 38px; }
        .led {                                                  /* Piloto apagado: gris oscuro */
            display: block; width: 14px; height: 14px; border-radius: 50%;
            margin: 0 auto 4px; background: #45475a;
            box-shadow: inset 0 0 3px rgba(0,0,0,0.6);
            transition: background 0.3s, box-shadow 0.3s;
        }
        .led.on { background: #a6e3a1; box-shadow: 0 0 8px 2px rgba(166,227,161,0.7); }   /* Piloto encendido: verde con brillo */
        .led.parpadeo { animation: parpadeo 1.2s infinite; }    /* Verde titilando: el dato es el ultimo que se recordo, no es en vivo */
        @keyframes parpadeo { 50% { opacity: 0.3; } }

        .data-box { background: #181825; padding: 14px; margin: 12px 0; border-radius: 8px; }
        .data-label { font-size: 0.9em; color: #a6adc8; }
        .data-value { font-weight: bold; color: #a6e3a1; font-size: 2em; display: block; margin-top: 4px; }
        .data-value.temp { color: #fab387; }                    /* La temperatura se ve en naranja */
        .status { font-size: 0.95em; color: #f9e2af; min-height: 1.2em; margin-top: 4px; }
        .viejo .data-value { opacity: 0.55; }                   /* Valores atenuados cuando son el ultimo estado recordado */
        .progress { height: 8px; background: #181825; border-radius: 4px; overflow: hidden; margin-top: 10px; }
        .progress > div { height: 100%; width: 0%; background: #89b4fa; transition: width 0.5s linear; }   /* Barra que avanza suave */
        .row { display: flex; justify-content: space-between; padding: 8px 0; border-bottom: 1px solid #45475a; }
        .row:last-child { border-bottom: 0; }
        .row b { color: #cdd6f4; }
        .row span { color: #a6adc8; }
        label { display: block; text-align: left; margin-top: 12px; }
        label b { color: #a6e3a1; }
        input[type="range"] { width: 100%; margin: 8px 0 0; }
        button {
            background-color: #89b4fa; color: #11111b; border: none; padding: 11px 20px;
            border-radius: 6px; cursor: pointer; font-weight: bold; font-size: 1em;
            transition: background 0.2s; margin: 8px 0 0; width: 100%;
        }
        button:hover:not(:disabled) { background-color: #b4befe; }
        button:disabled { opacity: 0.4; cursor: not-allowed; }  /* Boton apagado cuando no se puede usar */
        .btn-stop { background-color: #f38ba8; }                /* Boton Detener en rojo */
        .btn-stop:hover:not(:disabled) { background-color: #f5a3b8; }
        #msg { font-size: 0.85em; color: #a6adc8; min-height: 1.2em; margin-top: 10px; }
    </style>
</head>
<body>
<div class="wrap">

    <!-- TARJETA 1: titulo y tres pilotos de estado a la derecha -->
    <div class="card head">
        <h2>Control del horno</h2>
        <div class="pilotos">
            <div class="piloto" title="Verde: hay conexion con el broker (o con el ESP32 en modo local)">
                <span class="led" id="ledLink"></span><span id="txtLink">Broker</span>   <!-- Piloto 1: enlace -->
            </div>
            <div class="piloto" title="Verde: el horno esta enviando datos en este momento">
                <span class="led" id="ledDatos"></span>Datos                              <!-- Piloto 2: horno con datos -->
            </div>
            <div class="piloto" title="Verde: el horno esta en ciclo. Titila si es el ultimo estado recordado (sin datos en vivo)">
                <span class="led" id="ledCiclo"></span>Ciclo                              <!-- Piloto 3: horno funcionando -->
            </div>
        </div>
    </div>

    <!-- TARJETA 2: monitoreo (muestra el ultimo estado recordado si el horno se desconecta) -->
    <div class="card" id="cardMon">
        <h3>Monitoreo</h3>
        <div class="status" id="status">Estado: --</div>      <!-- En ejecucion / Detenido -->
        <div class="status" id="power">Potencia: -- %</div>   <!-- Potencia de calefaccion -->

        <div class="data-box">
            <span class="data-label">Temperatura actual</span>
            <span class="data-value temp" id="temp">-- °C</span>
        </div>

        <div class="data-box">
            <span class="data-label">Tiempo restante</span>
            <span class="data-value" id="timer">--:--:--</span>
            <div class="progress"><div id="bar"></div></div>   <!-- Barra de progreso del ciclo -->
        </div>
    </div>

    <!-- TARJETA 3: registro del ultimo ciclo -->
    <div class="card">
        <h3>Registro del último ciclo</h3>
        <div class="row"><span>Setpoint de temperatura</span><b id="cycTemp">--</b></div>
        <div class="row"><span>Setpoint de tiempo</span><b id="cycTime">--</b></div>
        <div class="row"><span>Resultado</span><b id="cycRes">--</b></div>
    </div>

    <!-- TARJETA 4: configurar y enviar un ciclo (limites: 100-250 °C y 1-240 min) -->
    <div class="card">
        <h3>Configurar ciclo</h3>

        <label>Temperatura: <b id="tempVal">150 °C</b></label>
        <input type="range" id="tempSlider" min="100" max="250" value="150" oninput="updateSliderText()">

        <label>Tiempo: <b id="timeVal">1 h</b></label>
        <input type="range" id="timeSlider" min="1" max="240" value="60" oninput="updateSliderText()">

        <button id="btnSend" onclick="enviarCiclo()" disabled>Enviar e iniciar</button>
        <button id="btnStop" class="btn-stop" onclick="detener()" disabled>Detener</button>
        <div id="msg"></div>   <!-- Mensajes al usuario (enviado, error, etc.) -->
    </div>

</div>

<script>
    // ===== BLOQUE JS 1: CONFIGURACIÓN (solo se usa cuando la página está en GitHub, modo MQTT) =====
    const BROKER_URL = "wss://bd8a2c7f.ala.us-east-1.emqxsl.com:8084/mqtt";   // Broker por WebSocket seguro (los navegadores no hablan MQTT directo)
    const MQTT_USER  = "web_horno";          // Usuario de la web en el broker (queda visible en la pagina: darle permisos minimos)
    const MQTT_PASS  = "web_horno";
    const TOPIC_CMD    = "horno/cmd";        // Donde la web envia ordenes
    const TOPIC_ESTADO = "horno/estado";     // De donde la web lee el estado
    const SIN_DATOS_MS = 15000;   // sin mensajes del horno por este tiempo = sin conexión
    const CLAVE_MEMORIA = "horno_ultimo_estado";   // Nombre bajo el cual el navegador guarda la memoria interna
    // ==================================================================================

    // ===== BLOQUE JS 2: MODO DE COMUNICACION =====
    // Si la página la sirve el ESP32 (http://192.168.4.1) habla directo con él por HTTP.
    // Si viene de GitHub (https) o de un archivo local, usa MQTT a través del broker.
    const LOCAL = location.protocol === "http:";   // true si la sirve el ESP32
    const ENLACE = LOCAL ? "Local" : "Broker";     // Texto que se muestra bajo el piloto 1

    const $ = id => document.getElementById(id);   // Atajo para buscar elementos por id
    let linkOk = false;                            // true si el enlace (HTTP o MQTT) funciona
    let client = null;                             // Cliente MQTT (solo modo broker)
    let slidersInit = false;                       // Para cargar los sliders una sola vez con el ultimo ciclo
    let ultRestante = 0;                           // Ultimo tiempo restante mostrado (se congela si el horno se desconecta)
    let rx = { online: false, active: false, temp: null, remaining: 0,     // Ultimo estado recibido (o recordado) del horno
               cycTemp: 0, cycMin: 0, result: "IDLE", power: 0,
               at: 0,                // momento en que llego el ultimo mensaje (cualquiera, sirve para detectar "sin datos")
               ultimo: 0,            // momento del ultimo dato REAL del horno (se guarda en la memoria)
               tieneDatos: false };  // true si hay algun estado, recibido ahora o recordado de antes

    // ===== BLOQUE JS 3: FORMATO DE TEXTOS =====
    function fmtMin(m) {                           // Minutos -> "45 min", "1 h", "2 h 30 min"
        m = parseInt(m, 10);
        if (m < 60) return m + " min";
        const h = Math.floor(m / 60), r = m % 60;
        return h + " h" + (r ? " " + r + " min" : "");
    }
    function fmtTiempo(seg) {                      // Segundos -> "HH:MM:SS"
        seg = Math.max(0, Math.floor(seg));
        const h = Math.floor(seg / 3600), m = Math.floor((seg % 3600) / 60), s = seg % 60;
        return String(h).padStart(2, "0") + ":" + String(m).padStart(2, "0") + ":" + String(s).padStart(2, "0");
    }
    function fmtHace(ms) {                         // Milisegundos -> "hace 20 s", "hace 5 min", "hace 2 h"
        const s = Math.max(0, Math.floor(ms / 1000));
        if (s < 60) return "hace " + s + " s";
        if (s < 3600) return "hace " + Math.floor(s / 60) + " min";
        return "hace " + Math.floor(s / 3600) + " h";
    }
    function updateSliderText() {                  // Muestra el valor de los sliders al moverlos
        $("tempVal").innerText = $("tempSlider").value + " °C";
        $("timeVal").innerText = fmtMin($("timeSlider").value);
    }
    function mensaje(t) { $("msg").innerText = t; }   // Escribe un aviso bajo los botones

    // ===== BLOQUE JS 4: MEMORIA INTERNA (recuerda el ultimo estado del horno) =====
    // Se guarda en el navegador (localStorage). Cada celular o PC tiene su propia memoria.
    function guardarMemoria() {                    // Se llama cada vez que llega un dato real del horno
        try {
            localStorage.setItem(CLAVE_MEMORIA, JSON.stringify({
                active: rx.active, temp: rx.temp, remaining: rx.remaining,
                cycTemp: rx.cycTemp, cycMin: rx.cycMin, result: rx.result,
                power: rx.power, ultimo: rx.ultimo
            }));
        } catch (e) { /* si el navegador no deja guardar, la pagina sigue funcionando sin memoria */ }
    }
    function cargarMemoria() {                     // Se llama una sola vez al abrir la pagina
        try {
            const g = localStorage.getItem(CLAVE_MEMORIA);
            if (!g) return;                        // No hay nada recordado todavia
            const m = JSON.parse(g);
            rx.active    = !!m.active;             // Recupera si el horno estaba en ciclo o no
            rx.temp      = (typeof m.temp === "number") ? m.temp : null;
            rx.remaining = m.remaining || 0;
            rx.cycTemp   = m.cycTemp || 0;
            rx.cycMin    = m.cycMin || 0;
            rx.result    = m.result || "IDLE";
            rx.power     = m.power || 0;
            rx.ultimo    = m.ultimo || 0;
            ultRestante  = rx.active ? rx.remaining : 0;   // Tiempo restante congelado en el valor recordado
            rx.tieneDatos = true;                  // Hay un estado para mostrar (aunque todavia no sea en vivo)
            if (rx.cycTemp > 0 && rx.cycMin > 0) { // Los sliders arrancan con el ultimo ciclo
                $("tempSlider").value = rx.cycTemp;
                $("timeSlider").value = rx.cycMin;
                slidersInit = true;
            }
        } catch (e) { /* memoria danada: se ignora */ }
    }

    // ===== BLOQUE JS 5: RECEPCION DEL ESTADO =====
    // ---------- Estado recibido (mismo JSON por MQTT y por HTTP) ----------
    function onEstado(d) {
        rx.online = d.online !== false;            // online:false lo manda el Last Will del broker si el ESP32 se cae
        if (rx.online) {
            rx.active    = !!d.active;             // Ciclo activo
            rx.temp      = (typeof d.temp === "number") ? d.temp : null;   // null si el sensor no da lectura
            rx.remaining = d.remaining_s || 0;     // Segundos restantes
            rx.cycTemp   = d.cycle_temp || 0;      // Datos del ultimo ciclo
            rx.cycMin    = d.cycle_min || 0;
            rx.result    = d.result || "IDLE";
            rx.power     = d.power || 0;
            rx.ultimo    = Date.now();             // Anota cuando llego este dato real
            rx.tieneDatos = true;
            guardarMemoria();                      // Lo recuerda por si despues se pierde la conexion o se cierra la pagina

            // La primera vez, los sliders toman los valores del último ciclo
            if (!slidersInit && rx.cycTemp > 0 && rx.cycMin > 0) {
                $("tempSlider").value = rx.cycTemp;
                $("timeSlider").value = rx.cycMin;
                updateSliderText();
                slidersInit = true;
            }
        }
        // Si online=false NO se borra nada: se sigue viendo el ultimo estado recordado
        rx.at = Date.now();                        // Anota cuando llego el mensaje (para detectar "sin datos")
        render();                                  // Redibuja la pantalla
    }

    // ===== BLOQUE JS 6: TRANSPORTE LOCAL (HTTP) =====
    // ---------- Transporte local (HTTP directo al ESP32) ----------
    function pollLocal() {
        fetch("/status", { cache: "no-store" })    // Pide el estado al ESP32
            .then(r => r.json())
            .then(d => { linkOk = true; onEstado(d); })      // Si responde: enlace OK y procesa el estado
            .catch(() => { linkOk = false; render(); });     // Si falla: enlace caido
    }

    // ===== BLOQUE JS 7: TRANSPORTE MQTT (broker EMQX) =====
    // ---------- Transporte MQTT (broker EMQX) ----------
    function conectarMqtt() {
        client = mqtt.connect(BROKER_URL, {
            username: MQTT_USER,
            password: MQTT_PASS,
            clientId: "web_" + Math.random().toString(16).slice(2, 10),   // ID aleatorio para no chocar con otras pestañas
            reconnectPeriod: 3000,                 // Reintenta cada 3 s si se cae
            connectTimeout: 10000,
            clean: true
        });
        client.on("connect", () => {               // Al conectar...
            linkOk = true;
            client.subscribe(TOPIC_ESTADO, { qos: 1 });   // ...se suscribe al estado del horno
            render();
        });
        client.on("reconnect", () => { linkOk = false; render(); });
        client.on("close",     () => { linkOk = false; render(); });
        client.on("error", err => mensaje("Error de conexión: " + err.message));
        client.on("message", (topic, payload) => {  // Cada mensaje de estado que llega
            try { onEstado(JSON.parse(payload.toString())); }
            catch (e) { console.error("JSON inválido", e); }
        });
    }

    function iniciarTransporte() {                 // Elige el camino segun donde esta la pagina
        if (LOCAL) {
            setInterval(pollLocal, 1000);          // Modo local: consulta cada 1 s
            pollLocal();
        } else {
            const s = document.createElement("script");   // Modo broker: carga la libreria mqtt.js desde internet
            s.src = "https://cdn.jsdelivr.net/npm/mqtt@5.10.1/dist/mqtt.min.js";
            s.onload = conectarMqtt;               // Cuando carga, conecta
            s.onerror = () => mensaje("No se pudo cargar la librería MQTT. ¿Hay internet?");
            document.head.appendChild(s);
        }
    }

    // ===== BLOQUE JS 8: ENVIO DE COMANDOS =====
    // ---------- Envío de comandos (mismo JSON por los dos caminos) ----------
    function enviar(obj, texto) {
        if (!linkOk) { mensaje("Sin conexión"); return; }   // No envia si no hay enlace
        if (LOCAL) {
            fetch("/cmd", {                        // Local: POST directo al ESP32
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify(obj)
            })
            .then(r => mensaje(r.ok ? texto : "El horno rechazó el comando"))
            .catch(() => mensaje("Error al enviar"));
        } else {
            // Los comandos NO van retenidos, para que no se repitan al reconectar
            client.publish(TOPIC_CMD, JSON.stringify(obj), { qos: 1, retain: false }, err => {
                mensaje(err ? "Error al enviar" : texto);
            });
        }
    }

    function enviarCiclo() {                       // Boton "Enviar e iniciar"
        const t = parseInt($("tempSlider").value, 10);
        const m = parseInt($("timeSlider").value, 10);
        if (rx.active && !confirm("Hay un ciclo en curso. ¿Reiniciarlo con " + t + " °C y " + fmtMin(m) + "?")) return;   // Pide confirmacion si ya hay un ciclo
        enviar({ cmd: "start", temp: t, min: m }, "Enviado: " + t + " °C, " + fmtMin(m));
    }
    function detener() {                           // Boton "Detener"
        enviar({ cmd: "stop" }, "Enviado: detener");
    }

    // ===== BLOQUE JS 9: DIBUJO DE LA PANTALLA =====
    // ---------- Pantalla ----------
    const RESULTADOS = { IDLE: "Sin ciclos aún", RUNNING: "En curso", DONE: "Completado", STOPPED: "Detenido", ERROR: "Falla del sensor" };   // Traduce el resultado del ESP32

    function render() {
        const fresco  = rx.at > 0 && (Date.now() - rx.at) < SIN_DATOS_MS;   // Llego un dato hace menos de 15 s?
        const hornoOk = rx.online && fresco;                                 // Horno en linea y con datos recientes (en vivo)
        const enMemoria = rx.tieneDatos && !hornoOk;                         // Se esta mostrando el ultimo estado recordado

        // --- Los tres pilotos ---
        $("txtLink").innerText = ENLACE;                                     // Piloto 1: "Broker" o "Local"
        $("ledLink").className  = "led" + (linkOk ? " on" : "");             // Verde si hay enlace
        $("ledDatos").className = "led" + (hornoOk ? " on" : "");            // Piloto 2: verde si el horno envia datos ahora
        $("ledCiclo").className = "led" + ((rx.tieneDatos && rx.active) ? " on" : "") + (enMemoria && rx.active ? " parpadeo" : "");   // Piloto 3: verde si esta en ciclo; titila si es dato recordado

        // --- Estado y potencia (si no hay datos en vivo, se ve el ultimo estado) ---
        let estado = "--";
        if (rx.tieneDatos) {
            estado = rx.active ? "En ejecución" : "Detenido";
            if (enMemoria) estado += " (último estado" + (rx.ultimo > 0 ? ", " + fmtHace(Date.now() - rx.ultimo) : "") + ")";   // Aclara que no es en vivo
        }
        $("status").innerText = "Estado: " + estado;
        $("power").innerText = "Potencia: " + (rx.tieneDatos ? (rx.active ? rx.power : 0) + " %" : "-- %");

        $("temp").innerText = rx.tieneDatos ? (rx.temp !== null ? rx.temp.toFixed(1) + " °C" : "Sin lectura") : "-- °C";

        // Tiempo restante: en vivo se interpola entre mensajes para que baje cada segundo; sin datos en vivo queda congelado
        let restante = ultRestante;
        if (hornoOk) {
            restante = rx.active ? Math.max(0, rx.remaining - (Date.now() - rx.at) / 1000) : 0;
            ultRestante = restante;                                          // Guarda el valor para congelarlo si se pierde la conexion
        }
        $("timer").innerText = rx.tieneDatos ? fmtTiempo(restante) : "--:--:--";

        const total = rx.cycMin * 60;                                        // Duracion total del ciclo en segundos
        const prog = (rx.tieneDatos && rx.active && total > 0) ? Math.min(100, Math.max(0, 100 * (1 - restante / total))) : 0;   // % transcurrido
        $("bar").style.width = prog + "%";

        $("cardMon").classList.toggle("viejo", enMemoria);                   // Atenua los valores si son recordados

        // Registro del último ciclo
        $("cycTemp").innerText = rx.cycTemp > 0 ? rx.cycTemp + " °C" : "--";
        $("cycTime").innerText = rx.cycMin > 0 ? fmtMin(rx.cycMin) : "--";
        $("cycRes").innerText  = rx.tieneDatos ? (RESULTADOS[rx.result] || rx.result) : "--";

        const puede = linkOk && hornoOk;                                     // Solo se puede operar con enlace y horno en vivo
        $("btnSend").disabled = !puede;
        $("btnStop").disabled = !(puede && rx.active);                       // Detener solo si hay ciclo activo
    }

    // ===== BLOQUE JS 10: ARRANQUE =====
    cargarMemoria();               // Recupera el ultimo estado recordado (si existe)
    setInterval(render, 500);      // Redibuja 2 veces por segundo (mantiene fluido el contador)
    updateSliderText();            // Muestra los valores iniciales de los sliders
    render();                      // Primer dibujo
    iniciarTransporte();           // Empieza a comunicarse (HTTP o MQTT)
</script>
</body>
</html>
