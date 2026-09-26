
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SROS // INTEGRIDADE TOTAL DO TRAJE ESPACIAL (TRÍADE MHD+SAMS+SROS)</title>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; user-select: none; }
        body, html { width: 100%; height: 100%; overflow: hidden; background-color: #020617; font-family: 'JetBrains Mono', 'Courier New', monospace; color: #00e5ff; }

        #canvas-container { width: 100%; height: 100%; position: absolute; top: 0; left: 0; z-index: 1; }
        canvas { display: block; width: 100%; height: 100%; }

        /* PAINEL TÁTICO ESQUERDO - TELEMETRIA DE INTEGRIDADE */
        .hud-left {
            position: absolute; top: 15px; left: 15px; z-index: 10;
            background: rgba(4, 15, 23, 0.92); border: 1px solid #00e5ff;
            border-radius: 8px; padding: 15px; width: 360px; max-height: calc(100vh - 120px);
            overflow-y: auto; box-shadow: 0 0 25px rgba(0, 229, 255, 0.2);
            backdrop-filter: blur(10px);
        }

        .hud-title {
            font-size: 12px; font-weight: 900; letter-spacing: 1.5px;
            text-transform: uppercase; color: #38bdf8; margin-bottom: 10px;
            border-bottom: 2px solid #00e5ff; padding-bottom: 6px;
            display: flex; justify-content: space-between; align-items: center;
        }

        .badge-system {
            background: #0284c7; color: #f0f9ff; font-size: 9px; padding: 2px 6px; border-radius: 3px; font-weight: bold;
        }

        .section-label {
            font-size: 10px; font-weight: bold; color: #64748b; margin: 10px 0 5px 0; text-transform: uppercase; letter-spacing: 0.8px;
        }

        /* GRID DE MÉTRICAS */
        .telemetry-grid {
            display: grid; grid-template-columns: 1fr 1fr; gap: 6px; margin-bottom: 12px;
        }
        .tele-box {
            background: rgba(15, 23, 42, 0.8); border: 1px solid rgba(0, 229, 255, 0.3);
            border-radius: 4px; padding: 6px; font-size: 9px;
        }
        .tele-lbl { color: #64748b; font-weight: bold; }
        .tele-val { color: #00ff66; font-size: 12px; font-weight: bold; margin-top: 2px; font-family: monospace; }
        .tele-val.alert { color: #ff3333; }

        /* BARRAS DE ZONA DE TRAJE */
        .zone-bar-container { margin-bottom: 6px; }
        .zone-header { display: flex; justify-content: space-between; font-size: 10px; margin-bottom: 2px; }
        .zone-bar { width: 100%; height: 6px; background: rgba(255,255,255,0.1); border-radius: 3px; overflow: hidden; }
        .zone-fill { height: 100%; background: #00ff66; width: 100%; transition: width 0.3s ease, background-color 0.3s ease; }

        /* PAINEL DIREITO - ATUAÇÃO DA TRÍADE */
        .hud-right {
            position: absolute; top: 15px; right: 15px; z-index: 10;
            background: rgba(4, 15, 23, 0.92); border: 1px solid #c084fc;
            border-radius: 8px; padding: 15px; width: 340px;
            box-shadow: 0 0 25px rgba(192, 132, 252, 0.2); backdrop-filter: blur(10px);
        }

        .status-card {
            background: rgba(15, 23, 42, 0.8); border-left: 3px solid #00ff66;
            padding: 8px; margin-bottom: 8px; border-radius: 0 4px 4px 0; font-size: 9.5px;
        }
        .status-card.mhd { border-color: #38bdf8; }
        .status-card.sams { border-color: #c084fc; }
        .status-card.sros { border-color: #00ff66; }

        .card-header { font-weight: bold; font-size: 10px; display: flex; justify-content: space-between; margin-bottom: 4px; }

        /* PAINEL INFERIOR DE SIMULAÇÕES E EVENTOS */
        .controls-bottom {
            position: absolute; bottom: 15px; left: 50%; transform: translateX(-50%); z-index: 10;
            display: flex; gap: 10px; width: 90%; max-width: 900px; justify-content: center;
        }

        .btn-sim {
            flex: 1; background: rgba(7, 25, 34, 0.95); border: 1px solid #00e5ff; color: #00e5ff;
            padding: 10px 12px; font-family: inherit; font-size: 10px; font-weight: bold;
            cursor: pointer; text-shadow: 0 0 3px #00e5ff; box-shadow: 0 0 12px rgba(0, 229, 255, 0.15);
            border-radius: 6px; transition: all 0.2s ease; text-transform: uppercase;
        }
        .btn-sim:hover { background: #00e5ff; color: #020617; text-shadow: none; transform: translateY(-2px); }
        .btn-sim.danger { border-color: #ff3333; color: #ff3333; text-shadow: 0 0 3px #ff3333; }
        .btn-sim.danger:hover { background: #ff3333; color: #ffffff; }
        .btn-sim.repair { border-color: #00ff66; color: #00ff66; text-shadow: 0 0 3px #00ff66; }
        .btn-sim.repair:hover { background: #00ff66; color: #020617; }

        /* BANNER INFERIOR DE LOGS DE IA */
        .hud-banner {
            position: absolute; bottom: 65px; left: 50%; transform: translateX(-50%); z-index: 10;
            background: rgba(2, 6, 23, 0.9); border: 1px solid #38bdf8; color: #e2e8f0;
            padding: 8px 20px; border-radius: 20px; font-size: 10px; font-weight: bold;
            letter-spacing: 0.8px; text-align: center; width: 80%; max-width: 800px;
            box-shadow: 0 0 15px rgba(56, 189, 248, 0.2);
        }

        .hud-left::-webkit-scrollbar { width: 4px; }
        .hud-left::-webkit-scrollbar-thumb { background: #00e5ff; }
    </style>
</head>
<body>

    <div id="canvas-container">
        <canvas id="suitCanvas"></canvas>
    </div>

    <!-- TELEMETRIA DE INTEGRIDADE GLOBAL -->
    <div class="hud-left">
        <div class="hud-title">
            <span>SUIT INTEGRITY // SROS DIAGNOSTIC</span>
            <span class="badge-system">EVA v4.8</span>
        </div>

        <div class="section-label">MÉTRICAS GERAIS EM TEMPO REAL</div>
        <div class="telemetry-grid">
            <div class="tele-box">
                <div class="tele-lbl">INTEGRIDADE TOTAL</div>
                <div class="tele-val" id="val-integrity">100.0%</div>
            </div>
            <div class="tele-box">
                <div class="tele-lbl">SELAMENTO MHD</div>
                <div class="tele-val" id="val-mhd-seal">99.8%</div>
            </div>
            <div class="tele-box">
                <div class="tele-lbl">ABSORÇÃO SAMS</div>
                <div class="tele-val" id="val-sams-damp">0.0 dB</div>
            </div>
            <div class="tele-box">
                <div class="tele-lbl">VARREDURA SROS</div>
                <div class="tele-val" id="val-sros-scan">0.02 ms</div>
            </div>
            <div class="tele-box">
                <div class="tele-lbl">PRESSÃO INTERNA</div>
                <div class="tele-val" id="val-pressure">4.30 PSI</div>
            </div>
            <div class="tele-box">
                <div class="tele-lbl">RESERVA NANO-REPARO</div>
                <div class="tele-val" id="val-repair-res">98.5%</div>
            </div>
        </div>

        <div class="section-label">INTEGRIDADE POR MATRIZ REGIONAL</div>

        <div class="zone-bar-container">
            <div class="zone-header"><span>HELMET / CAPACETE RADIAL</span><span id="lbl-z-head">100%</span></div>
            <div class="zone-bar"><div class="zone-fill" id="bar-z-head"></div></div>
        </div>
        <div class="zone-bar-container">
            <div class="zone-header"><span>TORSO & MATRIZ PNEUMÁTICA</span><span id="lbl-z-torso">100%</span></div>
            <div class="zone-bar"><div class="zone-fill" id="bar-z-torso"></div></div>
        </div>
        <div class="zone-bar-container">
            <div class="zone-header"><span>MEMBROS SUPERIORES (BRAÇOS)</span><span id="lbl-z-arms">100%</span></div>
            <div class="zone-bar"><div class="zone-fill" id="bar-z-arms"></div></div>
        </div>
        <div class="zone-bar-container">
            <div class="zone-header"><span>MEMBROS INFERIORES (PERNAS)</span><span id="lbl-z-legs">100%</span></div>
            <div class="zone-bar"><div class="zone-fill" id="bar-z-legs"></div></div>
        </div>
    </div>

    <!-- PAINEL DIREITO - STATUS DA TRÍADE -->
    <div class="hud-right">
        <div style="font-size: 11px; font-weight: bold; color: #c084fc; border-bottom: 1px solid #c084fc; padding-bottom: 5px; margin-bottom: 10px;">
            TRÍADE DE ATUAÇÃO E DEFESA
        </div>

        <div class="status-card mhd">
            <div class="card-header" style="color:#38bdf8;">
                <span>1. CAMPO MAGNETOIDRODINÂMICO (MHD)</span>
                <span id="st-mhd">ESTÁVEL</span>
            </div>
            Selamento magnético de micro-plasma contra vazamentos de vácuo e barreira térmica atômica.
        </div>

        <div class="status-card sams">
            <div class="card-header" style="color:#c084fc;">
                <span>2. ESCUDO SONICO ESTRUTURAL (SAMS)</span>
                <span id="st-sams">MODO VIGILÂNCIA</span>
            </div>
            Ressonância molecular em tempo real. Dissipa ondas de choque kineticas e impacto mecânico.
        </div>

        <div class="status-card sros">
            <div class="card-header" style="color:#00ff66;">
                <span>3. PROCESSADOR NEURAL QUÂNTICO (SROS)</span>
                <span id="st-sros">VARREDURA 100%</span>
            </div>
            Detecção antecipada de trincas, redistribuição pneumática instantânea e auto-cura.
        </div>
    </div>

    <!-- BANNER DE LOG DE EVENTOS DA IA -->
    <div class="hud-banner" id="banner-log">
        SISTEMA SROS: MONITORANDO INTEGRIDADE ESTRUTURAL DO TRAJE. NENHUMA ANOMALIA DETECTADA.
    </div>

    <!-- CONTROLES TÁTICOS DE SIMULAÇÃO -->
    <div class="controls-bottom">
        <button class="btn-sim danger" onclick="triggerEvent('meteorite')">Impacto Micrometeorito</button>
        <button class="btn-sim danger" onclick="triggerEvent('thermal')">Pico Térmico / Plasma</button>
        <button class="btn-sim danger" onclick="triggerEvent('pressure')">Fissura de Pressão</button>
        <button class="btn-sim repair" onclick="triggerEvent('repair')">Auto-Reparo da Tríade</button>
    </div>

    <script>
        const canvas = document.getElementById('suitCanvas');
        const ctx = canvas.getContext('2d');

        // SINTETIZADOR DE ÁUDIO WEB PARA ALERTAS TÁTICOS
        const AudioCtx = window.AudioContext || window.webkitAudioContext;
        let audioCtx = null;

        function playBeep(freq = 440, type = 'sine', duration = 0.15, vol = 0.05) {
            try {
                if (!audioCtx) audioCtx = new AudioCtx();
                if (audioCtx.state === 'suspended') audioCtx.resume();
                const osc = audioCtx.createOscillator();
                const gain = audioCtx.createGain();
                osc.type = type;
                osc.frequency.setValueAtTime(freq, audioCtx.currentTime);
                gain.gain.setValueAtTime(vol, audioCtx.currentTime);
                gain.gain.exponentialRampToValueAtTime(0.0001, audioCtx.currentTime + duration);
                osc.connect(gain);
                gain.connect(audioCtx.destination);
                osc.start();
                osc.stop(audioCtx.currentTime + duration);
            } catch(e) {}
        }

        function resize() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        }
        window.addEventListener('resize', resize);
        resize();

        // ESTADO DA INTEGRIDADE DO TRAJE POR ZONAS (0 a 100%)
        let zones = {
            head: 100,
            torso: 100,
            arms: 100,
            legs: 100
        };

        let activeEffects = {
            mhdFieldPulse: 0,
            samsWaveRadius: 0,
            samsWaveMax: 0,
            samsActive: false,
            laserScanY: 0,
            particles: []
        };

        let cycleTime = 0;

        function getOverallIntegrity() {
            return (zones.head * 0.25 + zones.torso * 0.35 + zones.arms * 0.2 + zones.legs * 0.2);
        }

        function triggerEvent(type) {
            const banner = document.getElementById('banner-log');

            if (type === 'meteorite') {
                // Impacto cinético no capacete/torso
                zones.head = Math.max(42, zones.head - 38);
                zones.torso = Math.max(55, zones.torso - 25);
                activeEffects.samsWaveRadius = 10;
                activeEffects.samsWaveMax = 180;
                activeEffects.samsActive = true;

                banner.innerText = "ALERTA SROS: IMPACTO KINÉTICO DETECTADO! ESCUDO SONICO SAMS DISVIPOU 88% DA ENERGIA.";
                document.getElementById('st-sams').innerText = "DISPERSÃO ATIVA";
                playBeep(880, 'sawtooth', 0.3, 0.08);

            } else if (type === 'thermal') {
                // Surto térmico nos braços e torso
                zones.arms = Math.max(35, zones.arms - 45);
                zones.torso = Math.max(60, zones.torso - 20);
                activeEffects.mhdFieldPulse = 1.0;

                banner.innerText = "ALERTA SROS: FLUXO RADIATIVO TÉRMICO! SELAMENTO MHD GEROU BARREIRA MAGNETO-PLASMÁTICA.";
                document.getElementById('st-mhd').innerText = "BLOQUEIO DE PLASMA";
                playBeep(600, 'square', 0.25, 0.08);

            } else if (type === 'pressure') {
                // Microfissura de pressão nas pernas
                zones.legs = Math.max(28, zones.legs - 52);
                
                // Cria partículas de fuga de O2 no ponto da fissura
                for (let i = 0; i < 30; i++) {
                    activeEffects.particles.push({
                        x: canvas.width / 2 - 30 + (Math.random() - 0.5) * 20,
                        y: canvas.height / 2 + 110 + (Math.random() - 0.5) * 20,
                        vx: (Math.random() - 0.5) * 4,
                        vy: (Math.random() - 0.5) * 4,
                        life: 1.0,
                        color: '#ff3333'
                    });
                }

                banner.innerText = "CRÍTICO SROS: FISSURA PNEUMÁTICA NA PERNA! CONTRA-MEDIDA MHD INJETANDO GEL MAGNETO-ESTANQUE.";
                playBeep(220, 'sawtooth', 0.4, 0.1);

            } else if (type === 'repair') {
                // Sequência de auto-cura da Tríade
                zones.head = 100;
                zones.torso = 100;
                zones.arms = 100;
                zones.legs = 100;
                activeEffects.mhdFieldPulse = 0.8;

                // Partículas de nano-reparo verdes
                for (let i = 0; i < 50; i++) {
                    activeEffects.particles.push({
                        x: canvas.width / 2 + (Math.random() - 0.5) * 120,
                        y: canvas.height / 2 + (Math.random() - 0.5) * 300,
                        vx: (Math.random() - 0.5) * 1.5,
                        vy: -Math.random() * 2,
                        life: 1.0,
                        color: '#00ff66'
                    });
                }

                banner.innerText = "SROS REPAIR: TRÍADE (MHD+SAMS+SROS) RESTAUROU A INTEGRIDADE NOMINAL DO TRAJE EM 100%.";
                document.getElementById('st-mhd').innerText = "ESTÁVEL";
                document.getElementById('st-sams').innerText = "MODO VIGILÂNCIA";
                document.getElementById('st-sros').innerText = "VARREDURA 100%";
                playBeep(523.25, 'sine', 0.2, 0.05);
            }

            updateUI();
        }

        function updateUI() {
            const total = getOverallIntegrity();
            const valIntegrity = document.getElementById('val-integrity');
            valIntegrity.innerText = total.toFixed(1) + "%";

            if (total < 60) {
                valIntegrity.className = "tele-val alert";
            } else {
                valIntegrity.className = "tele-val";
            }

            // Atualização das Barras de Zona
            const updateZone = (idFill, idLbl, val) => {
                const fill = document.getElementById(idFill);
                const lbl = document.getElementById(idLbl);
                fill.style.width = val + "%";
                lbl.innerText = Math.round(val) + "%";

                if (val < 50) {
                    fill.style.backgroundColor = "#ff3333";
                    lbl.style.color = "#ff3333";
                } else if (val < 80) {
                    fill.style.backgroundColor = "#eab308";
                    lbl.style.color = "#eab308";
                } else {
                    fill.style.backgroundColor = "#00ff66";
                    lbl.style.color = "#00ff66";
                }
            };

            updateZone('bar-z-head', 'lbl-z-head', zones.head);
            updateZone('bar-z-torso', 'lbl-z-torso', zones.torso);
            updateZone('bar-z-arms', 'lbl-z-arms', zones.arms);
            updateZone('bar-z-legs', 'lbl-z-legs', zones.legs);

            // Valores de suporte dinâmico
            document.getElementById('val-mhd-seal').innerText = (90 + (zones.torso * 0.1)).toFixed(1) + "%";
            document.getElementById('val-sams-damp').innerText = activeEffects.samsActive ? "42.8 dB" : "0.0 dB";
            document.getElementById('val-pressure').innerText = (3.50 + (zones.torso * 0.008)).toFixed(2) + " PSI";
        }

        // COORDENADAS DO CHASSI DO TRAJE
        function getSuitNodes(centerX, centerY) {
            return {
                head: { x: centerX, y: centerY - 150, r: 28 },
                neck: { x: centerX, y: centerY - 115 },
                shoulderL: { x: centerX - 55, y: centerY - 100 },
                shoulderR: { x: centerX + 55, y: centerY - 100 },
                elbowL: { x: centerX - 75, y: centerY - 30 },
                elbowR: { x: centerX + 75, y: centerY - 30 },
                wristL: { x: centerX - 65, y: centerY + 35 },
                wristR: { x: centerX + 65, y: centerY + 35 },
                hipL: { x: centerX - 30, y: centerY + 35 },
                hipR: { x: centerX + 30, y: centerY + 35 },
                kneeL: { x: centerX - 35, y: centerY + 120 },
                kneeR: { x: centerX + 35, y: centerY + 120 },
                ankleL: { x: centerX - 32, y: centerY + 200 },
                ankleR: { x: centerX + 32, y: centerY + 200 }
            };
        }

        function drawSuit() {
            const centerX = canvas.width / 2;
            const centerY = canvas.height / 2 + 10;
            cycleTime += 0.03;

            const n = getSuitNodes(centerX, centerY);

            // 1. DESENHAR GRID DE FUNDO E RADAR SROS
            ctx.strokeStyle = 'rgba(0, 229, 255, 0.05)';
            ctx.lineWidth = 1;
            for (let i = -300; i <= 300; i += 40) {
                ctx.beginPath(); ctx.moveTo(centerX + i, centerY - 250); ctx.lineTo(centerX + i, centerY + 250); ctx.stroke();
                ctx.beginPath(); ctx.moveTo(centerX - 300, centerY + i); ctx.lineTo(centerX + 300, centerY + i); ctx.stroke();
            }

            // 2. ESCUDO SONICO SAMS (ONDA RESSONANTE EXPANSIVA)
            if (activeEffects.samsWaveRadius > 0) {
                activeEffects.samsWaveRadius += 4;
                ctx.strokeStyle = `rgba(192, 132, 252, ${1 - (activeEffects.samsWaveRadius / activeEffects.samsWaveMax)})`;
                ctx.lineWidth = 3;
                ctx.beginPath();
                ctx.arc(centerX, centerY - 20, activeEffects.samsWaveRadius, 0, Math.PI * 2);
                ctx.stroke();

                if (activeEffects.samsWaveRadius >= activeEffects.samsWaveMax) {
                    activeEffects.samsWaveRadius = 0;
                    activeEffects.samsActive = false;
                }
            }

            // 3. ATUAÇÃO DO CAMPO MAGNETOIDRODINÂMICO (AURA MHD)
            if (activeEffects.mhdFieldPulse > 0.05 || zones.torso < 100) {
                activeEffects.mhdFieldPulse *= 0.98;
                const alpha = Math.max(0.1, activeEffects.mhdFieldPulse);
                ctx.shadowColor = '#38bdf8';
                ctx.shadowBlur = 20;
                ctx.strokeStyle = `rgba(56, 189, 248, ${alpha})`;
                ctx.lineWidth = 2;
                ctx.beginPath();
                ctx.ellipse(centerX, centerY, 100, 220, 0, 0, Math.PI * 2);
                ctx.stroke();
                ctx.shadowBlur = 0;
            }

            // 4. ESTRUTURA DO CORPO / PLACAS DO TRAJE POR CORES DE DANO
            const getColor = (val) => {
                if (val < 50) return '#ff3333';
                if (val < 80) return '#eab308';
                return '#00e5ff';
            };

            // Desenhar Malha do Capacete
            ctx.strokeStyle = getColor(zones.head);
            ctx.lineWidth = 2.5;
            ctx.beginPath();
            ctx.arc(n.head.x, n.head.y, n.head.r, 0, Math.PI * 2);
            ctx.stroke();

            // Viseira do Capacete
            ctx.fillStyle = 'rgba(56, 189, 248, 0.2)';
            ctx.beginPath();
            ctx.ellipse(n.head.x, n.head.y - 3, 18, 12, 0, 0, Math.PI * 2);
            ctx.fill(); ctx.stroke();

            // Torso e Placas Peitorais
            ctx.strokeStyle = getColor(zones.torso);
            ctx.beginPath();
            ctx.moveTo(n.shoulderL.x, n.shoulderL.y);
            ctx.lineTo(n.shoulderR.x, n.shoulderR.y);
            ctx.lineTo(n.hipR.x, n.hipR.y);
            ctx.lineTo(n.hipL.x, n.hipL.y);
            ctx.closePath();
            ctx.stroke();

            // Preenchimento Suave do Tronco
            ctx.fillStyle = zones.torso < 60 ? 'rgba(255, 51, 51, 0.08)' : 'rgba(0, 229, 255, 0.05)';
            ctx.fill();

            // Braços (Membros Superiores)
            ctx.strokeStyle = getColor(zones.arms);
            ctx.beginPath();
            ctx.moveTo(n.shoulderL.x, n.shoulderL.y); ctx.lineTo(n.elbowL.x, n.elbowL.y); ctx.lineTo(n.wristL.x, n.wristL.y);
            ctx.moveTo(n.shoulderR.x, n.shoulderR.y); ctx.lineTo(n.elbowR.x, n.elbowR.y); ctx.lineTo(n.wristR.x, n.wristR.y);
            ctx.stroke();

            // Pernas (Membros Inferiores)
            ctx.strokeStyle = getColor(zones.legs);
            ctx.beginPath();
            ctx.moveTo(n.hipL.x, n.hipL.y); ctx.lineTo(n.kneeL.x, n.kneeL.y); ctx.lineTo(n.ankleL.x, n.ankleL.y);
            ctx.moveTo(n.hipR.x, n.hipR.y); ctx.lineTo(n.kneeR.x, n.kneeR.y); ctx.lineTo(n.ankleR.x, n.ankleR.y);
            ctx.stroke();

            // NÓS ARTICULARES DA TRÍADE SROS
            Object.keys(n).forEach(key => {
                const node = n[key];
                ctx.fillStyle = '#ffffff';
                ctx.beginPath();
                ctx.arc(node.x, node.y, 3, 0, Math.PI * 2);
                ctx.fill();
            });

            // 5. FEIXE DE VARREDURA LAZER QUÂNTICA SROS
            activeEffects.laserScanY += 3;
            if (activeEffects.laserScanY > 480) activeEffects.laserScanY = 0;
            const scanY = (centerY - 220) + activeEffects.laserScanY;

            ctx.strokeStyle = 'rgba(0, 255, 102, 0.7)';
            ctx.lineWidth = 1.5;
            ctx.beginPath();
            ctx.moveTo(centerX - 130, scanY);
            ctx.lineTo(centerX + 130, scanY);
            ctx.stroke();

            // Efeito Brilho do Laser
            ctx.fillStyle = 'rgba(0, 255, 102, 0.15)';
            ctx.fillRect(centerX - 130, scanY - 4, 260, 8);

            // 6. GERENCIAMENTO E DESENHO DE PARTÍCULAS
            for (let i = activeEffects.particles.length - 1; i >= 0; i--) {
                let p = activeEffects.particles[i];
                p.x += p.vx;
                p.y += p.vy;
                p.life -= 0.02;

                ctx.fillStyle = p.color;
                ctx.globalAlpha = Math.max(0, p.life);
                ctx.fillRect(p.x, p.y, 2.5, 2.5);
                ctx.globalAlpha = 1.0;

                if (p.life <= 0) {
                    activeEffects.particles.splice(i, 1);
                }
            }
        }

        function render() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // Fundo Hangar Tático
            ctx.fillStyle = '#020617';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            drawSuit();

            requestAnimationFrame(render);
        }

        // INICIALIZAÇÃO
        updateUI();
        render();
    </script>
</body>
</html>
