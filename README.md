<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Control del horno</title>
    <style>
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
        .wrap { width: 380px; max-width: 94vw; margin: 16px; }
        .card {
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
        .badge.ok  { background: #a6e3a1; color: #11111b; }
        .badge.bad { background: #f38ba8; color: #11111b; }
        .data-box { background: #181825; padding: 14px; margin: 12px 0; border-radius: 8px; }
        .data-label { font-size: 0.9em; color: #a6adc8; }
        .data-value { font-weight: bold; color: #a6e3a1; font-size: 2em; display: block; margin-top: 4px; }
        .data-value.temp { color: #fab387; }
        .status { font-size: 0.95em; color: #f9e2af; min-height: 1.2em; margin-top: 4px; }
        .progress { height: 8px; background: #181825; border-radius: 4px; overflow: hidden; margin-top: 10px; }
        .progress > div { height: 100%; width: 0%; background: #89b4fa; transition: width 0.5s linear; }
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
        button:disabled { opacity: 0.4; cursor: not-allowed; }
        .btn-stop { background-color: #f38ba8; }
        .btn-stop:hover:not(:disabled) { background-color: #f5a3b8; }
        #msg { font-size: 0.85em; color: #a6adc8; min-height: 1.2em; margin-top: 10px; }
    </style>
</head>
<body>
<div class="wrap">

    <div class="card">
        <h2>Control del horno</h2>
        <div class="badges">
            <span class="badge" id="badgeBroker">Broker: conectando…</span>
            <span class="badge" id="badgeHorno">Horno: sin datos</span>
        </div>
    </div>

    <div class="card">
        <h3>Monitoreo</h3>
        <div class="status" id="status">Estado: --</div>

        <div class="data-box">
            <span class="data-label">Temperatura actual</span>
            <span class="data-value temp" id="temp">-- °C</span>
        </div>

        <div class="data-box">
            <span class="data-label">Tiempo restante</span>
            <span class="data-value" id="timer">--:--:--</span>
            <div class="progress"><div id="bar"></div></div>
        </div>
    </div>

    <div class="card">
        <h3>Registro del último ciclo</h3>
        <div class="row"><span>Setpoint de temperatura</span><b id="cycTemp">--</b></div>
        <div class="row"><span>Setpoint de tiempo</span><b id="cycTime">--</b></div>
        <div class="row"><span>Resultado</span><b id="cycRes">--</b></div>
    </div>

    <div class="card">
        <h3>Configurar ciclo</h3>

        <label>Temperatura: <b id="tempVal">150 °C</b></label>
        <input type="range" id="tempSlider" min="1" max="450" value="150" oninput="updateSliderText()">

        <label>Tiempo: <b id="timeVal">1 h</b></label>
        <input type="range" id="timeSlider" min="1" max="120" value="60" oninput="updateSliderText()">

        <button id="btnSend" onclick="enviarCiclo()" disabled>Enviar e iniciar</button>
        <button id="btnStop" class="btn-stop" onclick="detener()" disabled>Detener</button>
        <div id="msg"></div>
    </div>

</div>

<script src="https://cdn.jsdelivr.net/npm/mqtt@5.10.1/dist/mqtt.min.js"></script>
<script>
    // ===================== CONFIGURACIÓN =====================
    const BROKER_URL = "wss://bd8a2c7f.ala.us-east-1.emqxsl.com:8084/mqtt";
    const MQTT_USER  = "web_horno";
    const MQTT_PASS  = "web_horno";
    const TOPIC_CMD    = "horno/cmd";
    const TOPIC_ESTADO = "horno/estado";
    const SIN_DATOS_MS = 15000;   // sin mensajes del horno por este tiempo = sin conexión
    // =========================================================

    const $ = id => document.getElementById(id);
    let brokerOk = false;
    let slidersInit = false;
    let rx = { online: false, active: false, temp: null, remaining: 0,
               cycTemp: 0, cycMin: 0, result: "IDLE", at: 0 };

    function fmtMin(m) {
        m = parseInt(m, 10);
        if (m < 60) return m + " min";
        const h = Math.floor(m / 60), r = m % 60;
        return h + " h" + (r ? " " + r + " min" : "");
    }
    function fmtTiempo(seg) {
        seg = Math.max(0, Math.floor(seg));
        const h = Math.floor(seg / 3600), m = Math.floor((seg % 3600) / 60), s = seg % 60;
        return String(h).padStart(2, "0") + ":" + String(m).padStart(2, "0") + ":" + String(s).padStart(2, "0");
    }
    function updateSliderText() {
        $("tempVal").innerText = $("tempSlider").value + " °C";
        $("timeVal").innerText = fmtMin($("timeSlider").value);
    }
    function mensaje(t) { $("msg").innerText = t; }

    // ---------- MQTT ----------
    const client = mqtt.connect(BROKER_URL, {
        username: MQTT_USER,
        password: MQTT_PASS,
        clientId: "web_" + Math.random().toString(16).slice(2, 10),
        reconnectPeriod: 3000,
        connectTimeout: 10000,
        clean: true
    });

    client.on("connect", () => {
        brokerOk = true;
        client.subscribe(TOPIC_ESTADO, { qos: 1 });
        render();
    });
    client.on("reconnect", () => { brokerOk = false; render(); });
    client.on("close",     () => { brokerOk = false; render(); });
    client.on("error", err => mensaje("Error de conexión: " + err.message));

    client.on("message", (topic, payload) => {
        try {
            const d = JSON.parse(payload.toString());
            rx.online = d.online !== false;
            if (rx.online) {
                rx.active    = !!d.active;
                rx.temp      = (typeof d.temp === "number") ? d.temp : null;
                rx.remaining = d.remaining_s || 0;
                rx.cycTemp   = d.cycle_temp || 0;
                rx.cycMin    = d.cycle_min || 0;
                rx.result    = d.result || "IDLE";

                // La primera vez, los sliders toman los valores del último ciclo
                if (!slidersInit && rx.cycTemp > 0 && rx.cycMin > 0) {
                    $("tempSlider").value = rx.cycTemp;
                    $("timeSlider").value = rx.cycMin;
                    updateSliderText();
                    slidersInit = true;
                }
            }
            rx.at = Date.now();
            render();
        } catch (e) { console.error("JSON inválido", e); }
    });

    function enviar(obj, texto) {
        if (!brokerOk) { mensaje("Sin conexión con el broker"); return; }
        // Los comandos NO van retenidos, para que no se repitan al reconectar
        client.publish(TOPIC_CMD, JSON.stringify(obj), { qos: 1, retain: false }, err => {
            mensaje(err ? "Error al enviar" : texto);
        });
    }

    // ---------- Acciones ----------
    function enviarCiclo() {
        const t = parseInt($("tempSlider").value, 10);
        const m = parseInt($("timeSlider").value, 10);
        if (rx.active && !confirm("Hay un ciclo en curso. ¿Reiniciarlo con " + t + " °C y " + fmtMin(m) + "?")) return;
        enviar({ cmd: "start", temp: t, min: m }, "Enviado: " + t + " °C, " + fmtMin(m));
    }
    function detener() {
        enviar({ cmd: "stop" }, "Enviado: detener");
    }

    // ---------- Pantalla ----------
    const RESULTADOS = { IDLE: "Sin ciclos aún", RUNNING: "En curso", DONE: "Completado", STOPPED: "Detenido" };

    function render() {
        const fresco  = rx.at > 0 && (Date.now() - rx.at) < SIN_DATOS_MS;
        const hornoOk = rx.online && fresco;

        const bb = $("badgeBroker");
        bb.className = "badge " + (brokerOk ? "ok" : "bad");
        bb.innerText = brokerOk ? "Broker: conectado" : "Broker: desconectado";

        const bh = $("badgeHorno");
        if (rx.at === 0)  { bh.className = "badge";        bh.innerText = "Horno: sin datos"; }
        else if (hornoOk) { bh.className = "badge ok";     bh.innerText = "Horno: en línea"; }
        else              { bh.className = "badge bad";    bh.innerText = "Horno: sin conexión"; }

        $("status").innerText = "Estado: " + (hornoOk ? (rx.active ? "En ejecución" : "Detenido") : "--");

        $("temp").innerText = (hornoOk && rx.temp !== null) ? rx.temp.toFixed(1) + " °C"
                            : (hornoOk ? "Sin lectura" : "-- °C");

        // Tiempo restante, interpolado entre mensajes para que baje cada segundo
        let restante = 0;
        if (hornoOk && rx.active) restante = rx.remaining - (Date.now() - rx.at) / 1000;
        $("timer").innerText = hornoOk ? fmtTiempo(restante) : "--:--:--";

        const total = rx.cycMin * 60;
        const prog = (hornoOk && rx.active && total > 0) ? Math.min(100, Math.max(0, 100 * (1 - restante / total))) : 0;
        $("bar").style.width = prog + "%";

        // Registro del último ciclo
        $("cycTemp").innerText = rx.cycTemp > 0 ? rx.cycTemp + " °C" : "--";
        $("cycTime").innerText = rx.cycMin > 0 ? fmtMin(rx.cycMin) : "--";
        $("cycRes").innerText  = hornoOk ? (RESULTADOS[rx.result] || rx.result) : "--";

        const puede = brokerOk && hornoOk;
        $("btnSend").disabled = !puede;
        $("btnStop").disabled = !(puede && rx.active);
    }

    setInterval(render, 500);
    updateSliderText();
    render();
</script>
</body>
</html>
