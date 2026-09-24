<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SROS // DIAGNOSTIC TOTAL ARMOR SUIT</title>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }
        body, html { width: 100%; height: 100%; overflow: hidden; background-color: #040608; font-family: 'Courier New', Courier, monospace; color: #00e5ff; }
        #canvas-container { width: 100%; height: 100%; position: absolute; top: 0; left: 0; z-index: 1; }
        canvas { display: block; width: 100%; height: 100%; }
        .hud-overlay { position: absolute; top: 20px; left: 20px; z-index: 10; pointer-events: none; text-shadow: 0 0 5px #00e5ff; background: rgba(4, 10, 14, 0.9); padding: 15px; border: 1px solid #00e5ff; border-radius: 4px; box-shadow: 0 0 15px rgba(0, 229, 255, 0.2); max-width: 320px; }
        h1 { font-size: 14px; margin-bottom: 8px; border-bottom: 1px solid #00e5ff; padding-bottom: 4px; text-transform: uppercase; }
        .telemetry-item { font-size: 11px; margin: 5px 0; display: flex; justify-content: space-between; }
        .status-active { color: #00ff66; animation: blink 1.5s infinite; }
        .status-alert { color: #ff3333; animation: blink 0.5s infinite; }
        .btn-action { position: absolute; bottom: 20px; left: 20px; z-index: 10; background: #071922; border: 1px solid #00e5ff; color: #00e5ff; padding: 10px 20px; font-family: inherit; font-size: 11px; cursor: pointer; text-shadow: 0 0 3px #00e5ff; box-shadow: 0 0 10px rgba(0,229,255,0.1); border-radius: 4px; pointer-events: auto; }
        .btn-action:hover { background: #00e5ff; color: #040608; font-weight: bold; }
        .legend { position: absolute; top: 20px; right: 20px; z-index: 10; background: rgba(4, 10, 14, 0.9); border: 1px solid #00e5ff; padding: 10px; font-size: 10px; border-radius: 4px; }
        .legend-item { margin: 4px 0; display: flex; align-items: center; }
        .dot { width: 8px; height: 8px; border-radius: 50%; margin-right: 8px; display: inline-block; }
        @keyframes blink { 0%, 100% { opacity: 1; } 50% { opacity: 0.4; } }
    </style>
</head>
<body>

    <div id="canvas-container">
        <canvas id="totalSuitCanvas"></canvas>
    </div>

    <div class="hud-overlay">
        <h1>SROS // FULL SUIT DIAGNOSTIC</h1>
        <div class="telemetry-item"><span>VARREDURA FÍSICA:</span> <span id="status-scan" class="status-active">INTEGRIDADE NOMINAL</span></div>
        <div class="telemetry-item"><span>MALHA ESTRUTURAL:</span> <span id="armor-val">100%</span></div>
        <div class="telemetry-item"><span>CINETICA DE MEMBROS:</span> <span>CONECTADO</span></div>
        <div class="telemetry-item"><span>POTÊNCIA DO REATOR:</span> <span id="power-val">1.21 GW</span></div>
        <div class="telemetry-item"><span>CONSUMO ENERGIA:</span> <span id="dreno-val">2.2 kW/s</span></div>
    </div>

    <div class="legend">
        <div class="legend-item"><span class="dot" style="background:#00e5ff;"></span>Chassi Externo do Traje (Mapeamento)</div>
        <div class="legend-item"><span class="dot" style="background:#00ff66;"></span>Fluxo de Energia / Impulso Interno</div>
        <div class="legend-item"><span class="dot" style="background:#ff3333;"></span>Sobrecarga nos Atuadores</div>
    </div>

    <button class="btn-action" id="trigger-overload">SOBRECARREGAR SISTEMA E CONVERSÃO</button>

    <script>
        const canvas = document.getElementById('totalSuitCanvas');
        const ctx = canvas.getContext('2d');

        function redimensionar() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        }
        window.addEventListener('resize', redimensionar);
        redimensionar();

        // Parâmetros do Sistema Completo
        let modoSobrecarga = false;
        let integridadeArmadura = 100.0;
        let potenciaReator = 1.21;
        let drenoEnergia = 2.2;
        let cicloAnimacao = 0;

        // Partículas que correm pelo interior de toda a extensão do traje
        const particulasEnergia = [];
        const maxParticulas = 80;

        // Estrutura geométrica do Traje Completo (Base proporcional para o desenho cibernético)
        function obterEsqueletoTraje(centerX, centerY) {
            return {
                capacete: { x: centerX, y: centerY - 140, r: 24 },
                pescoco: { x1: centerX, y1: centerY - 116, x2: centerX, y2: centerY - 105 },
                ombroE: { x: centerX - 45, y: centerY - 95 },
                ombroD: { x: centerX + 45, y: centerY - 95 },
                cotoveloE: { x: centerX - 60, y: centerY - 40 },
                cotoveloD: { x: centerX + 60, y: centerY - 40 },
                pulsoE: { x: centerX - 55, y: centerY + 15 },
                pulsoD: { x: centerX + 55, y: centerY + 15 },
                quadrilE: { x: centerX - 25, y: centerY + 30 },
                quadrilD: { x: centerX + 25, y: centerY + 30 },
                joelhoE: { x: centerX - 30, y: centerY + 100 },
                joelhoD: { x: centerX + 30, y: centerY + 100 },
                tornozeloE: { x: centerX - 28, y: centerY + 170 },
                tornozeloD: { x: centerX + 28, y: centerY + 170 }
            };
        }

        function criarParticulaEnergia(e) {
            // Sorteia um membro do corpo para a energia percorrer de forma interna
            const caminhos = [
                ['ombroE', 'cotoveloE', 'pulsoE'],
                ['ombroD', 'cotoveloD', 'pulsoD'],
                ['quadrilE', 'joelhoE', 'tornozeloE'],
                ['quadrilD', 'joelhoD', 'tornozeloD']
            ];
            const caminhoSorteado = caminhos[Math.floor(Math.random() * caminhos.length)];
            
            return {
                caminho: caminhoSorteado,
                nóAtual: 0,
                x: e[caminhoSorteado[0]].x,
                y: e[caminhoSorteado[0]].y,
                progresso: 0,
                velocidade: 0.02 + Math.random() * 0.03
            };
        }

        const armorEl = document.getElementById('armor-val');
        const powerEl = document.getElementById('power-val');
        const drenoEl = document.getElementById('dreno-val');
        const statusEl = document.getElementById('status-scan');

        function draw() {
            // Fundo escuro de hangar / laboratório espacial
            ctx.fillStyle = '#040608';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            const centerX = canvas.width / 2;
            const centerY = canvas.height / 2 + 10; // Leve deslocamento para baixo para centrar o corpo humano inteiro
            cicloAnimacao += 0.03;

            const e = obterEsqueletoTraje(centerX, centerY);

            // Gerenciamento contínuo das partículas internas de energia do chassi
            if (particulasEnergia.length < maxParticulas) {
                particulasEnergia.push(criarParticulaEnergia(e));
            }

            // Lógica física de sobrecarga nos conversores integrados
            if (modoSobrecarga) {
                potenciaReator = Math.min(4.85, potenciaReator + 0.05);
                drenoEnergia = Math.min(45.8, drenoEnergia + 1.2);
                integridadeArmadura = Math.max(76.2, integridadeArmadura - 0.2);

                statusEl.innerText = "ALERTA: SOBRECARGA CRÍTICA";
                statusEl.className = "status-alert";
            } else {
                potenciaReator = Math.max(1.21, potenciaReator - 0.1);
                drenoEnergia = Math.max(2.2, drenoEnergia - 0.9);
                if (integridadeArmadura < 100) integridadeArmadura += 0.05;

                statusEl.innerText = "INTEGRIDADE NOMINAL";
                statusEl.className = "status-active";
                statusEl.style.color = "#00ff66";
            }

            // ==========================================
            // 1. DESENHAR O CONTORNO COMPLETO EXTERNO DO TRAJE (CHASSI 2D VETORIAL)
            // ==========================================
            ctx.strokeStyle = modoSobrecarga ? 'rgba(255, 51, 51, 0.4)' : 'rgba(0, 229, 255, 0.3)';
            ctx.lineWidth = 2;
            ctx.lineJoin = "round";

            // Desenho da Viseira e Capacete Inteiro
            ctx.beginPath();
            ctx.arc(e.capacete.x, e.capacete.y, e.capacete.r, 0, Math.PI * 2);
            ctx.stroke();
            
            // Placa do Peito e Ombro a Ombro
            ctx.beginPath();
            ctx.moveTo(e.ombroE.x, e.ombroE.y);
            ctx.lineTo(e.ombroD.x, e.ombroD.y);
            ctx.lineTo(e.quadrilD.x, e.quadrilD.y);
            ctx.lineTo(e.quadrilE.x, e.quadrilE.y);
            ctx.closePath();
            ctx.stroke();

            // Braço Esquerdo Completo
            ctx.beginPath();
            ctx.moveTo(e.ombroE.x, e.ombroE.y);
            ctx.lineTo(e.cotoveloE.x, e.cotoveloE.y);
            ctx.lineTo(e.pulsoE.x, e.pulsoE.y);
            ctx.stroke();

            // Braço Direito Completo
            ctx.beginPath();
            ctx.moveTo(e.ombroD.x, e.ombroD.y);
            ctx.lineTo(e.cotoveloD.x, e.cotoveloD.y);
            ctx.lineTo(e.pulsoD.x, e.pulsoD.y);
            ctx.stroke();

            // Perna Esquerda Completa
            ctx.beginPath();
            ctx.moveTo(e.quadrilE.x, e.quadrilE.y);
            ctx.lineTo(e.joelhoE.x, e.joelhoE.y);
            ctx.lineTo(e.tornozeloE.x, e.tornozeloE.y);
            ctx.stroke();

            // Perna Direito Completa
            ctx.beginPath();
            ctx.moveTo(e.quadrilD.x, e.quadrilD.y);
            ctx.lineTo(e.joelhoD.x, e.joelhoD.y);
            ctx.lineTo(e.tornozeloD.x, e.tornozeloD.y);
            ctx.stroke();

            // Reator Torácico Central Circular Unificado (Estilo Homem de Ferro)
            ctx.strokeStyle = modoSobrecarga ? '#ff3333' : '#00e5ff';
            ctx.fillStyle = modoSobrecarga ? 'rgba(255,51,51,0.2)' : 'rgba(0,229,255,0.2)';
