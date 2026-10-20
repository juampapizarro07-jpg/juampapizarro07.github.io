# juampapizarro07.github.io
<!DOCTYPE html>
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
            align-items: center;
            min-height: 100vh;
            margin: 0;
        }
        .card {
            background-color: #313244;
            padding: 30px;
            border-radius: 12px;
            box-shadow: 0 4px 15px rgba(0,0,0,0.5);
            text-align: center;
            width: 350px;
            max-width: 90vw;
            margin: 16px;
        }
        h2 { margin-top: 0; color: #89b4fa; }
        .badges { display: flex; gap: 8px; justify-content: center; margin-bottom: 12px; flex-wrap: wrap; }
        .badge {
            font-size: 0.8em;
            padding: 4px 10px;
            border-radius: 20px;
            background: #45475a;
            color: #cdd6f4;
        }
        .badge.ok  { background: #a6e3a1; color: #11111b; }
        .badge.bad { background: #f38ba8; color: #11111b; }
        .data-box {
            background: #181825;
            padding: 15px;
            margin: 15px 0;
            border-radius: 8px;
            font-size: 1.2em;
        }
        .data-value { font-weight: bold; color: #a6e3a1; font-size: 1.6em; }
        .status-box { font-size: 0.95em; color: #f9e2af; margin-bottom: 15px; min-height: 1.2em; }
        input[type="range"] { width: 100%; margin: 10px 0; }
        button {
            background-color: #89b4fa;
            color: #11111b;
            border: none;
            padding: 10px 20px;
            border-radius: 6px;
            cursor: pointer;
            font-weight: bold;
            font-size: 1em;
            transition: background 0.2s;
            margin: 5px 0;
            width: 100%;
        }
        button:hover:not(:disabled) { background-color: #b4befe; }
        button:disabled { opacity: 0.4; cursor: not-allowed; }
        .btn-secondary { background-color: #fab387; }
        .btn-secondary:hover:not(:disabled) { background-color: #f9c9a8; }
        .btn-stop { background-color: #f38ba8; }
        .btn-stop:hover:not(:disabled) { background-color: #f5a3b8; }
        hr { border: 0; height: 1px; background: #45475a; margin: 20px 0; }
        #msg { font-size: 0.85em; color: #a6adc8; min-height: 1.2em; margin-top: 8px; }
        #tempBox { display: none; }
    </style>
</head>
<body>
    <div class="card">
        <h2>Control del horno</h2>

        <div class="badges">
            <span class="badge" id="badgeBroker">Broker: conectando…</span>
            <span class="badge" id="badgeHorno">Horno: sin datos</span>
        </div>

        <div class="status-box" id="status">Estado: --</div>

        <div class="data-box" id="tempBox">
            Temperatura:<br>
            <span id="temp" class="data-value">-- °C</span>
        </div>

        <div class="data-box">
            Tiempo restante:<br>
            <span id="timer" class="data-value">--:--</span>
        </div>

        <button id="btnAdd" onclick="addTime()" disabled>+ 30 segundos</button>
        <button id="btnStop" class="btn-stop" onclick="stopHorno()" disabled>Detener</button>

        <hr>

        <h3>Configurar duración</h3>
        <label>Duración: <span id="sliderVal">5 min</span></label>
        <input type="range" id="minSlider" min="1" max="240" value="5" oninput="updateSliderText(this.value)">
        <button id="btnSet" class="btn-secondary" onclick="setNewTimer()" disabled>Enviar e iniciar</button>

        <div id="msg"></div>
    </div>

    <script src="https://cdn.jsdelivr.net/npm/mqtt@5.10.1/dist/mqtt.min.js"></script>
    <script>
        // ===================== CONFIGURACIÓN =====================
        const BROKER_URL = "wss://bd8a2c7f.ala.us-east-1.emqxsl.com:8084/mqtt";
        const MQTT_USER  = "USUARIO_WEB";
        const MQTT_PASS  = "CLAVE_WEB";
        const TOPIC_CMD    = "horno/cmd";
        const TOPIC_ESTADO = "horno/estado";
        const SIN_DATOS_MS = 15000;
        // =========================================================

        const $ = id => document.getElementById(id);
        let brokerOk = false;
        let rx = { online: false, state: "IDLE", remaining: 0, total: 0, temp: null, reason: "", at: 0 };

        function fmtMin(m) {
            m = parseInt(m, 10);
            if (m < 60) return m + " min";
            const h = Math.floor(m / 60), r = m % 60;
            return h + " h" + (r ? " " + r + " min" : "");
        }
        function updateSliderText(v) { $("sliderVal").innerText = fmtMin(v); }

        function fmtTiempo(seg) {
            seg = Math.max(0, Math.floor(seg));
            const h = Math.floor(seg / 3600);
            const m = Math.floor((seg % 3600) / 60);
            const s = seg % 60;
            const mm = String(m).padStart(2, "0"), ss = String(s).padStart(2, "0");
            return h > 0 ? h + ":" + mm + ":" + ss : mm + ":" + ss;
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
        client.on("error", err => { mensaje("Error de conexión: " + err.message); });

        client.on("message", (topic, payload) => {
            try {
                const d = JSON.parse(payload.toString());
                rx.online    = d.online !== false;
                if (rx.online) {
                    rx.state     = d.state || "IDLE";
                    rx.remaining = d.remaining_s || 0;
                    rx.total     = d.total_s || 0;
                    rx.temp      = (typeof d.temp === "number") ? d.temp : null;
                    rx.reason    = d.reason || "";
                }
                rx.at = Date.now();
                render();
            } catch (e) { console.error("JSON inválido", e); }
        });

        function enviar(obj, texto) {
            if (!brokerOk) { mensaje("Sin conexión con el broker"); return; }
            client.publish(TOPIC_CMD, JSON.stringify(obj), { qos: 1, retain: false }, err => {
                mensaje(err ? "Error al enviar" : texto);
            });
        }

        // ---------- Acciones ----------
        function addTime()    { enviar({ add_sec: 30 }, "Enviado: +30 segundos"); }
        function stopHorno()  { enviar({ stop: true }, "Enviado: detener"); }
        function setNewTimer() {
            const m = parseInt($("minSlider").value, 10);
            enviar({ set_minutes: m }, "Enviado: iniciar " + fmtMin(m));
        }

        // ---------- Pantalla ----------
        function render() {
            const fresco = rx.at > 0 && (Date.now() - rx.at) < SIN_DATOS_MS;
            const hornoOk = rx.online && fresco;

            const bb = $("badgeBroker");
            bb.className = "badge " + (brokerOk ? "ok" : "bad");
            bb.innerText = brokerOk ? "Broker: conectado" : "Broker: desconectado";

            const bh = $("badgeHorno");
            if (rx.at === 0) { bh.className = "badge"; bh.innerText = "Horno: sin datos"; }
            else if (hornoOk) { bh.className = "badge ok"; bh.innerText = "Horno: en línea"; }
            else { bh.className = "badge bad"; bh.innerText = "Horno: sin conexión"; }

            const nombres = { IDLE: "En espera", RUNNING: "En curso", FINISHED: "Finalizado" };
            let st = "Estado: " + (hornoOk ? (nombres[rx.state] || rx.state) : "--");
            if (hornoOk && rx.state === "FINISHED" && rx.reason) st += " (" + rx.reason + ")";
            $("status").innerText = st;

            let restante = 0;
            if (hornoOk && rx.state === "RUNNING") {
                restante = rx.remaining - (Date.now() - rx.at) / 1000;
            }
            $("timer").innerText = hornoOk ? fmtTiempo(restante) : "--:--";

            if (hornoOk && rx.temp !== null) {
                $("tempBox").style.display = "block";
                $("temp").innerText = rx.temp.toFixed(1) + " °C";
            } else {
                $("tempBox").style.display = "none";
            }

            const puede = brokerOk && hornoOk;
            $("btnAdd").disabled  = !(puede && rx.state === "RUNNING");
            $("btnStop").disabled = !(puede && rx.state === "RUNNING");
            $("btnSet").disabled  = !puede;
        }

        setInterval(render, 500);
        updateSliderText($("minSlider").value);
        render();
    </script>
</body>
</html>
