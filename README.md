<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Simulador de Exploração Abissal</title>
    <style>
        :root {
            --bg-color: #020208;
            --panel-bg: rgba(10, 15, 30, 0.85);
            --border-color: #1a2342;
            --text-color: #c5d1ec;
            --hud-glow: #00ffcc;
        }

        body {
            margin: 0;
            padding: 20px;
            background-color: var(--bg-color);
            color: var(--text-color);
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            overflow-x: hidden;
        }

        h1 {
            font-size: 24px;
            margin-bottom: 5px;
            text-transform: uppercase;
            letter-spacing: 2px;
            text-shadow: 0 0 10px rgba(0, 255, 204, 0.4);
            color: #ffffff;
        }

        p.subtitle {
            margin: 0 0 20px 0;
            color: #5c6b94;
            font-size: 14px;
        }

        .container {
            display: flex;
            gap: 20px;
            max-width: 1100px;
            width: 100%;
            justify-content: center;
        }

        .canvas-container {
            position: relative;
            background: #000;
            border: 2px solid var(--border-color);
            border-radius: 8px;
            box-shadow: 0 15px 35px rgba(0,0,0,0.8);
        }

        canvas {
            display: block;
            background: linear-gradient(to bottom, #010b14 0%, #000205 70%);
        }

        .panel {
            background: var(--panel-bg);
            border: 1px solid var(--border-color);
            border-radius: 8px;
            padding: 20px;
            width: 280px;
            display: flex;
            flex-direction: column;
            box-sizing: border-box;
        }

        h2 {
            font-size: 16px;
            margin: 0 0 15px 0;
            color: var(--hud-glow);
            text-transform: uppercase;
            border-bottom: 1px solid var(--border-color);
            padding-bottom: 5px;
        }

        .telemetry-item {
            display: flex;
            justify-content: space-between;
            margin: 8px 0;
            font-family: 'Courier New', Courier, monospace;
            font-size: 13px;
        }

        .telemetry-item span:first-child {
            color: #7182b1;
        }

        .instructions {
            margin-top: auto;
            background: rgba(0,0,0,0.3);
            padding: 10px;
            border-radius: 4px;
            font-size: 12px;
            line-height: 1.5;
            border-left: 3px solid var(--hud-glow);
        }

        .key {
            background: #161f38;
            padding: 2px 6px;
            border-radius: 3px;
            border: 1px solid #314375;
            color: #fff;
            font-weight: bold;
        }
    </style>
</head>
<body>

    <h1>Simulador de Exploração Abissal</h1>
    <p class="subtitle">Física de Empuxo Fluido e Varredura Hidrodinâmica Sonar</p>

    <div class="container">
        <!-- ÁREA VISUAL DO CANVA -->
        <div class="canvas-container">
            <canvas id="subCanvas" width="700" height="500"></canvas>
        </div>

        <!-- PAINEL DE TELEMETRIA (HUD) -->
        <div class="panel">
            <h2>Telemetria Digital</h2>
            <div class="telemetry-item"><span>Profundidade:</span><span id="txtDepth" style="color:#fff">0 m</span></div>
            <div class="telemetry-item"><span>Pressão Hidrost.:</span><span id="txtPressure" style="color:#ff5555">0 atm</span></div>
            <div class="telemetry-item"><span>Empuxo/Estado:</span><span id="txtBuoyancy" style="color:var(--hud-glow)">Neutro</span></div>
            <div class="telemetry-item"><span>Velocidade Y:</span><span id="txtVelY">0.0 m/s</span></div>
            <div class="telemetry-item"><span>Velocidade X:</span><span id="txtVelX">0.0 m/s</span></div>
            <div class="telemetry-item"><span>Sonar Ativo:</span><span style="color:#39ff14">Ping ok</span></div>

            <div class="instructions">
                <strong>Controles de Propulsão:</strong><br><br>
                <span class="key">▲</span> Subir Lastro (Empuxo+)<br>
                <span class="key">▼</span> Inundar Tanque (Empuxo-)<br>
                <span class="key">◄</span> Propulsor Popa Esquerda<br>
                <span class="key">►</span> Propulsor Popa Direita<br>
            </div>
        </div>
    </div>

    <script>
        const canvas = document.getElementById('subCanvas');
        const ctx = canvas.getContext('2d');

        // --- SISTEMA FÍSICO DO SUBMARINO ---
        const sub = {
            x: 150,
            y: 100,
            vx: 0,
            vy: 0,
            largura: 50,
            altura: 25,
            massa: 1.2,
            empuxo: 0.12,          // Força vertical controlável (Default equilibra gravidade)
            gravidade: 0.12,       // Força descendente constante
            arrastoFator: 0.96,    // Atrito/viscosidade da água profunda
            propulsaoX: 0.15,
            potenciaLastro: 0.005,
            direcaoLook: 1         // 1 = Direita, -1 = Esquerda
        };

        // --- ENGENHARIA DE INPUTS (TECLADO) ---
        const keys = { ArrowUp: false, ArrowDown: false, ArrowLeft: false, ArrowRight: false };
        window.addEventListener('keydown', e => { if(e.key in keys) keys[e.key] = true; });
        window.addEventListener('keyup', e => { if(e.key in keys) keys[e.key] = false; });

        // --- ALGORITMO PROCEDURAL DE TERRENO (RELEVO DE TRINCHEIRA) ---
        const pontosTerreno = [];
        const totalPontos = 40;
        const espacamento = canvas.width / (totalPontos - 1);

        // Gera uma topografia irregular simulando falhas tectônicas no leito
        let alturaBase = 420;
        for (let i = 0; i < totalPontos; i++) {
            // Cria elevações e depressões usando ruído pseudo-aleatório acumulado
            if (i > 10 && i < 28) {
                alturaBase += (Math.random() * 25 - 10); // Declive para criar trincheira
            } else {
                alturaBase += (Math.random() * 16 - 8);
            }
            // Limita tetos máximos para o assoalho não colidir bizarramente no topo
            alturaBase = Math.max(340, Math.min(alturaBase, 480));
            pontosTerreno.push({ x: i * espacamento, y: alturaBase });
        }

        // --- SISTEMA DE PARTÍCULAS (MARINHA BIOLUMINESCENTE - NEVE MARINHA) ---
        const neveMarinha = [];
        for (let i = 0; i < 40; i++) {
            neveMarinha.push({
                x: Math.random() * canvas.width,
                y: Math.random() * canvas.height,
                raio: Math.random() * 1.5,
                velY: Math.random() * 0.3 + 0.1
            });
        }

        // --- LOOP CRONOLÓGICO PRINCIPAL DO MOTOR (60 FPS) ---
        function update() {
            // 1. Processamento da Dinâmica de Lastro e Empuxo
            if (keys.ArrowUp) {
                sub.empuxo -= sub.potenciaLastro; // Torna o submarino mais leve que a água (sobe)
            }
            if (keys.ArrowDown) {
                sub.empuxo += sub.potenciaLastro; // Captura água nos tanques (afunda)
            }
            
            // Estabilização automática de flutuação se não houver comandos ativos
            if (!keys.ArrowUp && !keys.ArrowDown) {
                sub.empuxo += (sub.gravidade - sub.empuxo) * 0.05;
            }

            // Limites estritos do sistema de flutuação hidráulica
            sub.empuxo = Math.max(0.08, Math.min(sub.empuxo, 0.16));

            // 2. Processamento da Propulsão Horizontal
            if (keys.ArrowRight) {
                sub.vx += sub.propulsaoX;
                sub.direcaoLook = 1;
            }
            if (keys.ArrowLeft) {
                sub.vx -= sub.propulsaoX;
                sub.direcaoLook = -1;
            }

            // 3. Aplicação do Modelo de Integração de Forças de Vetor e Arrasto
            // Força Líquida Y = Gravidade - Empuxo
            const forcaY = sub.gravidade - sub.empuxo;
            sub.vy += forcaY;

            // Fluidodinâmica: Aplicação de atrito viscoso viscosidade cinemática da água
            sub.vx *= sub.arrastoFator;
            sub.vy *= sub.arrastoFator;

            // Atualização geométrica posicional
            sub.x += sub.vx;
            sub.y += sub.vy;

            // 4. Barreira de Colisão Primal (Paredes Laterais e Superfície)
            if (sub.x < 0) sub.x = 0;
            if (sub.x > canvas.width - sub.largura) sub.x = canvas.width - sub.largura;
            if (sub.y < 10) { sub.y = 10; sub.vy = 0; }

            // 5. Interceptação Dinâmica de Colisão com o Terreno Procedural (Amostragem Linear)
            const indexTerreno = Math.floor(sub.x / espacamento);
            if (indexTerreno >= 0 && indexTerreno < totalPontos - 1) {
                const ptA = pontosTerreno[indexTerreno];
                const ptB = pontosTerreno[indexTerreno + 1];
                // Interpolação linear da altura exata do solo sob o submarino
                const t = (sub.x - ptA.x) / espacamento;
                const alturaSoloSobSub = ptA.y + t * (ptB.y - ptA.y);

                if (sub.y + sub.altura > alturaSoloSobSub) {
                    sub.y = alturaSoloSobSub - sub.altura;
                    sub.vy = 0; // Impacto mecânico absorvido pelo fundo
                }
            }

            // 6. Atualização de Telemetria e HUD Digital
            const profundidadeSimulada = Math.floor(sub.y * 18); // Fator de escala métrica
            const pressaoSimulada = Math.floor((profundidadeSimulada / 10) + 1);

            document.getElementById('txtDepth').innerText = `${profundidadeSimulada} metros`;
