
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SROS // FULL PRESSURE DIAGNOSTIC</title>
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
        <canvas id="pressureCanvas"></canvas>
    </div>

    <div class="hud-overlay">
        <h1>SROS // PRESSURE DIAGNOSTIC</h1>
        <div class="telemetry-item"><span>ESTADO COMPLEMENTAR:</span> <span id="status-scan" class="status-active">PRESSÃO ESTÁVEL</span></div>
        <div class="telemetry-item"><span>PRESSÃO INTERNA:</span> <span id="press-val">4.30 PSI</span></div>
        <div class="telemetry-item"><span>FLUXO DE O₂ INJETADO:</span> <span id="flow-val">0.45 L/min</span></div>
        <div class="telemetry-item"><span>ESTANQUEIDADE MALHA:</span> <span id="leak-val">100% SECURE</span></div>
        <div class="telemetry-item"><span>SUPRIMENTO DISPONÍVEL:</span> <span>96.4%</span></div>
    </div>

    <div class="legend">
        <div class="legend-item"><span class="dot" style="background:#00e5ff;"></span>Chassi do Traje Estabilizado</div>
        <div class="legend-item"><span class="dot" style="background:#3399ff;"></span>Injeção Interna Base de O₂</div>
        <div class="legend-item"><span class="dot" style="background:#ff3333;"></span>Vazamento Detectado no Membro</div>
    </div>

    <button class="btn-action" id="trigger-leak">SIMULAR MICROFISSURA NA PERNA ESQUERDA</button>

    <script>
        const canvas = document.getElementById('pressureCanvas');
        const ctx = canvas.getContext('2d');

        function redimensionar() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        }
        window.addEventListener('resize', redimensionar);
        redimensionar();

        // Parâmetros de Pressurização Pneumática Interna
        let vazamentoAtivo = false;
        let pressaoInterna = 4.30;
        let fluxoO2 = 0.45;
        let cicloPulso = 0;

        // Partículas que simulam a injeção gasosa de O2 expandindo dentro das pernas e braços
        const gasesInternos = [];
        const maxGases = 120;

        // Coordenadas estruturais do Chassi Completo
        function obterEstruturaTraje(centerX, centerY) {
            return {
                capacete: { x: centerX, y: centerY - 140, r: 24 },
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

        function criarParticulaGas(e) {
            const caminhos = [
                ['ombroE', 'cotoveloE', 'pulsoE'],
                ['ombroD', 'cotoveloD', 'pulsoD'],
                ['quadrilE', 'joelhoE', 'tornozeloE'],
                ['quadrilD', 'joelhoD', 'tornozeloD'],
                ['ombroE', 'capacete'],
                ['ombroD', 'capacete']
            ];
            
            let caminhoSorteado = caminhos[Math.floor(Math.random() * caminhos.length)];
            if (vazamentoAtivo && Math.random() > 0.3) {
                caminhoSorteado = ['quadrilE', 'joelhoE', 'tornozeloE'];
            }

            return {
                caminho: caminhoSorteado,
                nó: 0,
                x: e[caminhoSorteado[0]].x,
                y: e[caminhoSorteado[0]].y,
                progresso: 0,
                velocidade: 0.015 + Math.random() * 0.02
            };
        }

        const pressEl = document.getElementById('press-val');
        const flowEl = document.getElementById('flow-val');
        const leakEl = document.getElementById('leak-val');
        const statusEl = document.getElementById('status-scan');
        const btnLeak = document.getElementById('trigger-leak');

        // Evento do botão de simulação
        btnLeak.addEventListener('click', () => {
            vazamentoAtivo = !vazamentoAtivo;
            if (vazamentoAtivo) {
                btnLeak.innerText = "REPARAR MALHA / ESTANCAR VAZAMENTO";
                btnLeak.style.borderColor = "#ff3333";
                btnLeak.style.color = "#ff3333";
            } else {
                btnLeak.innerText = "SIMULAR MICROFISSURA NA PERNA ESQUERDA";
                btnLeak.style.borderColor = "#00e5ff";
                btnLeak.style.color = "#00e5ff";
            }
        });

        function draw() {
            // Hangar de diagnóstico escuro cibernético
            ctx.fillStyle = '#040608';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            const centerX = canvas.width / 2;
            const centerY = canvas.height / 2 + 10;
            cicloPulso += 0.04;

            const e = obterEstruturaTraje(centerX, centerY);

            // Gerencia partículas do fluxo gasoso interno
            if (gasesInternos.length < maxGases) {
                gasesInternos.push(criarParticulaGas(e));
            }

            // Física em tempo real da resposta do regulador pneumático do traje
            if (vazamentoAtivo) {
                pressaoInterna = Math.max(3.12, pressaoInterna - 0.015);
                fluxoO2 = Math.min(2.50, fluxoO2 + 0.04);

                statusEl.innerText = "CRÍTICO: DESPRESSURIZAÇÃO";
                statusEl.className = "status-alert";
                leakEl.innerText = "LEAK AT LEFT LEG";
                leakEl.style.color = "#ff3333";
            } else {
                pressaoInterna = Math.min(4.30, pressaoInterna + 0.02);
                fluxoO2 = Math.max(0.45, fluxoO2 - 0.05);

                statusEl.innerText = "PRESSÃO ESTÁVEL";
                statusEl.className = "status-active";
                statusEl.style.color = "#00ff66";
                leakEl.innerText = "100% SECURE";
                leakEl.style.color = "#00ff66";
            }

            // Atualização dos dados do painel HUD
            pressEl.innerText = pressaoInterna.toFixed(2) + " PSI";
            flowEl.innerText = fluxoO2.toFixed(2) + " L/min";

            // ==========================================
            // 1. DESENHAR O CONTORNO COMPLETO DO CHASSI EXTERNO
            // ==========================================
            ctx.lineWidth = 2;
            
            // Desenho do Tronco e Braços Padrão Ciano
            ctx.strokeStyle = 'rgba(0, 229, 255, 0.3)';
            ctx.beginPath();
            ctx.arc(e.capacete.x, e.capacete.y, e.capacete.r, 0, Math.PI * 2);
            ctx.moveTo(e.ombroE.x, e.ombroE.y); ctx.lineTo(e.ombroD.x, e.ombroD.y);
            ctx.lineTo(e.quadrilD.x, e.quadrilD.y); ctx.lineTo(e.quadrilE.x, e.quadrilE.y); ctx.closePath();
            ctx.moveTo(e.ombroE.x, e.ombroE.y); ctx.lineTo(e.cotoveloE.x, e.cotoveloE.y); ctx.lineTo(e.pulsoE.x, e.pulsoE.y);
            ctx.moveTo(e.ombroD.x, e.ombroD.y); ctx.lineTo(e.cotoveloD.x, e.cotoveloD.y); ctx.lineTo(e.pulsoD.x, e.pulsoD.y);
            ctx.moveTo(e.quadrilD.x, e.quadrilD.y); ctx.lineTo(e.joelhoD.x, e.joelhoD.y); ctx.lineTo(e.tornozeloD.x, e.tornozeloD.y);
            ctx.stroke();

            // Segmento Isolado: Perna Esquerda (Pisca em vermelho em caso de vazamento)
            ctx.strokeStyle = vazamentoAtivo && Math.floor(cicloPulso * 3) % 2 === 0 ? 'rgba(255, 51, 51, 1)' : 'rgba(0, 229, 255, 0.3)';
            ctx.beginPath();
            ctx.moveTo(e.quadrilE.x, e.quadrilE.y);
            ctx.lineTo(e.joelhoE.x, e.joelhoE.y);
            ctx.lineTo(e.tornozeloE.x, e.tornozeloE.y);
            ctx.stroke();

            // Nível Global de Brilho Pneumático Interno
            ctx.fillStyle = vazamentoAtivo ? 'rgba(255, 51, 51, 0.03)' : 'rgba(0, 229, 255, 0.03)';
            ctx.fillRect(centerX - 45, centerY - 90, 90, 115);

            // ==========================================
            // 2. LÓGICA DAS PARTÍCULAS GASOSAS INTERNAS DE COMPENSAÇÃO (O₂)
            // ==========================================
            for (let i = gasesInternos.length - 1; i >= 0; i--) {
                let p = gasesInternos[i];
                p.progresso += p.velocidade;

                if (p.progresso >= 1) {
                    p.progresso = 0;
                    p.nó++;
                }

                if (p.nó >= p.caminho.length - 1) {
                    gasesInternos.splice(i, 1);
                    continue;
                }

                let p1 = e[p.caminho[p.nó]];
                let p2 = e[p.caminho[p.nó + 1]];

                if (p1 && p2) {
                    p.x = p1.x + (p2.x - p1.x) * p.progresso;
                    p.y = p1.y + (p2.y - p1.y) * p.progresso;

                    const eNaPernaE = p.caminho.includes('joelhoE');
                    ctx.fillStyle = (vazamentoAtivo && eNaPernaE) ? '#ff3333' : '#3399ff';
                    ctx.shadowColor = ctx.fillStyle;
                    ctx.shadowBlur = 4;
                    ctx.beginPath();
                    ctx.arc(p.x, p.y, 2.5, 0, Math.PI * 2);
                    ctx.fill();
                    ctx.shadowBlur = 0;
                }
            }

            // ==========================================
            // 3. PONTOS DE ARTICULAÇÃO E EFEITO DE VAZAMENTO
            // ==========================================
            Object.keys(e).forEach(k => {
                let node = e[k];
                const isPernaE = k === 'quadrilE' || k === 'joelhoE' || k === 'tornozeloE';
                ctx.fillStyle = (vazamentoAtivo && isPernaE) ? '#ff3333' : '#00e5ff';
                ctx.beginPath();
                ctx.arc(node.x, node.y, k === 'capacete' ? 4 : 3, 0, Math.PI * 2);
                ctx.fill();
            });

            // Spray Físico de Vazamento na Perna Esquerda
            if (vazamentoAtivo) {
                const leakX = e.joelhoE.x;
                const leakY = e.joelhoE.y;

                ctx.strokeStyle = '#ff3333';
                ctx.lineWidth = 1.5;
                ctx.beginPath();
                ctx.arc(leakX, leakY, 8 + Math.sin(cicloPulso * 6) * 4, 0, Math.PI * 2);
                ctx.stroke();

                for (let k = 0; k < 4; k++) {
                    let angle = Math.PI * 0.8 + (Math.random() - 0.5) * 1.5;
                    let dist = Math.random() * 30 + 5;
                    ctx.fillStyle = 'rgba(255, 51, 51, ' + Math.random() + ')';
                    ctx.fillRect(leakX + Math.cos(angle) * dist, leakY + Math.sin(angle) * dist, 2, 2);
                }
            }

            requestAnimationFrame(draw);
        }

        draw();
    </script>
</body>
</html>
