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
        .badges { display: flex; gap: 8px; justify-content: center; flex-wrap: wrap; }
        .badge { font-size: 0.8em; padding: 4px 10px; border-radius: 20px; background: #45475a; color: #cdd6f4; }
        .badge.ok  { background: #a6e3a1; color: #11111b; }     /* Insignia verde: conectado */
        .badge.bad { background: #f38ba8; color: #11111b; }     /* Insignia roja: desconectado */
        .data-box { background: #181825; padding: 14px; margin: 12px 0; border-radius: 8px; }
        .data-label { font-size: 0.9em; color: #a6adc8; }
        .data-value { font-weight: bold; color: #a6e3a1; font-size: 2em; display: block; margin-top: 4px; }
        .data-value.temp { color: #fab387; }                    /* La temperatura se ve en naranja */
        .status { font-size: 0.95em; color: #f9e2af; min-height: 1.2em; margin-top: 4px; }
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

    <!-- TARJETA 1: titulo e insignias de conexion -->
    <div class="card">
        <h2>Control del horno</h2>
        <div class="badges">
            <span class="badge" id="badgeLink">Conectando…</span>      <!-- Estado del enlace (Local o Broker) -->
            <span class="badge" id="badgeHorno">Horno: sin datos</span> <!-- Estado del horno (en linea / sin conexion) -->
        </div>
    </div>

    <!-- TARJETA 2: monitoreo en vivo -->
    <div class="card">
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
    const MQTT_USER  = "web_horno";        // Usuario de la web en el broker (queda visible en la pagina: darle permisos minimos)
    const MQTT_PASS  = "web_horno";
    const TOPIC_CMD    = "horno/cmd";        // Donde la web envia ordenes
    const TOPIC_ESTADO = "horno/estado";     // De donde la web lee el estado
    const SIN_DATOS_MS = 15000;   // sin mensajes del horno por este tiempo = sin conexión
    // ==================================================================================

    // ===== BLOQUE JS 2: MODO DE COMUNICACION =====
    // Si la página la sirve el ESP32 (http://192.168.4.1) habla directo con él por HTTP.
    // Si viene de GitHub (https) o de un archivo local, usa MQTT a través del broker.
    const LOCAL = location.protocol === "http:";   // true si la sirve el ESP32
    const ENLACE = LOCAL ? "Local" : "Broker";     // Texto que se muestra en la insignia

    const $ = id => document.getElementById(id);   // Atajo para buscar elementos por id
    let linkOk = false;                            // true si el enlace (HTTP o MQTT) funciona
    let client = null;                             // Cliente MQTT (solo modo broker)
    let slidersInit = false;                       // Para cargar los sliders una sola vez con el ultimo ciclo
    let rx = { online: false, active: false, temp: null, remaining: 0,     // Ultimo estado recibido del horno
               cycTemp: 0, cycMin: 0, result: "IDLE", power: 0, at: 0 };   // "at" = momento en que llego

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
    function updateSliderText() {                  // Muestra el valor de los sliders al moverlos
        $("tempVal").innerText = $("tempSlider").value + " °C";
        $("timeVal").innerText = fmtMin($("timeSlider").value);
    }
    function mensaje(t) { $("msg").innerText = t; }   // Escribe un aviso bajo los botones

    // ===== BLOQUE JS 4: RECEPCION DEL ESTADO =====
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

            // La primera vez, los sliders toman los valores del último ciclo
            if (!slidersInit && rx.cycTemp > 0 && rx.cycMin > 0) {
                $("tempSlider").value = rx.cycTemp;
                $("timeSlider").value = rx.cycMin;
                updateSliderText();
                slidersInit = true;
            }
        }
        rx.at = Date.now();                        // Anota cuando llego el dato (para detectar "sin datos")
        render();                                  // Redibuja la pantalla
    }

    // ===== BLOQUE JS 5: TRANSPORTE LOCAL (HTTP) =====
    // ---------- Transporte local (HTTP directo al ESP32) ----------
    function pollLocal() {
        fetch("/status", { cache: "no-store" })    // Pide el estado al ESP32
            .then(r => r.json())
            .then(d => { linkOk = true; onEstado(d); })      // Si responde: enlace OK y procesa el estado
            .catch(() => { linkOk = false; render(); });     // Si falla: enlace caido
    }

    // ===== BLOQUE JS 6: TRANSPORTE MQTT (broker EMQX) =====
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

    // ===== BLOQUE JS 7: ENVIO DE COMANDOS =====
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

    // ===== BLOQUE JS 8: DIBUJO DE LA PANTALLA =====
    // ---------- Pantalla ----------
    const RESULTADOS = { IDLE: "Sin ciclos aún", RUNNING: "En curso", DONE: "Completado", STOPPED: "Detenido", ERROR: "Falla del sensor" };   // Traduce el resultado del ESP32

    function render() {
        const fresco  = rx.at > 0 && (Date.now() - rx.at) < SIN_DATOS_MS;   // Llego un dato hace menos de 15 s?
        const hornoOk = rx.online && fresco;                                 // Horno en linea y con datos recientes

        const bl = $("badgeLink");                                           // Insignia del enlace
        bl.className = "badge " + (linkOk ? "ok" : "bad");
        bl.innerText = ENLACE + (linkOk ? ": conectado" : ": desconectado");

        const bh = $("badgeHorno");                                          // Insignia del horno
        if (rx.at === 0)  { bh.className = "badge";     bh.innerText = "Horno: sin datos"; }
        else if (hornoOk) { bh.className = "badge ok";  bh.innerText = "Horno: en línea"; }
        else              { bh.className = "badge bad"; bh.innerText = "Horno: sin conexión"; }

        $("status").innerText = "Estado: " + (hornoOk ? (rx.active ? "En ejecución" : "Detenido") : "--");
        $("power").innerText = "Potencia: " + (hornoOk ? (rx.active ? rx.power : 0) + " %" : "-- %");

        $("temp").innerText = (hornoOk && rx.temp !== null) ? rx.temp.toFixed(1) + " °C"
                            : (hornoOk ? "Sin lectura" : "-- °C");

        // Tiempo restante, interpolado entre mensajes para que baje cada segundo
        let restante = 0;
        if (hornoOk && rx.active) restante = rx.remaining - (Date.now() - rx.at) / 1000;
        $("timer").innerText = hornoOk ? fmtTiempo(restante) : "--:--:--";

        const total = rx.cycMin * 60;                                        // Duracion total del ciclo en segundos
        const prog = (hornoOk && rx.active && total > 0) ? Math.min(100, Math.max(0, 100 * (1 - restante / total))) : 0;   // % transcurrido
        $("bar").style.width = prog + "%";

        // Registro del último ciclo
        $("cycTemp").innerText = rx.cycTemp > 0 ? rx.cycTemp + " °C" : "--";
        $("cycTime").innerText = rx.cycMin > 0 ? fmtMin(rx.cycMin) : "--";
        $("cycRes").innerText  = hornoOk ? (RESULTADOS[rx.result] || rx.result) : "--";

        const puede = linkOk && hornoOk;                                     // Solo se puede operar con enlace y horno en linea
        $("btnSend").disabled = !puede;
        $("btnStop").disabled = !(puede && rx.active);                       // Detener solo si hay ciclo activo
    }

    // ===== BLOQUE JS 9: ARRANQUE =====
    setInterval(render, 500);      // Redibuja 2 veces por segundo (mantiene fluido el contador)
    updateSliderText();            // Muestra los valores iniciales de los sliders
    render();                      // Primer dibujo
    iniciarTransporte();           // Empieza a comunicarse (HTTP o MQTT)
</script>
</body>
</html>
