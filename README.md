<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>射箭点击记录工具</title>
    <style>
        body { font-family: Arial; margin: 20px; }
        #canvas { border: 1px solid #ccc; cursor: crosshair; margin-top: 10px; }
        #stats { margin-top: 20px; }
        label { margin-right: 10px; }
        button { margin-right: 10px; margin-top: 10px; }
        table {
                font-size: 14px;
                border-collapse: collapse;
                width: 100%;
            }
            th, td {
                padding: 6px 10px;
                border: 1px solid #ccc;
                text-align: center;
            }
    </style>

    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body>
    <h3 style="text-align: center;">射箭点击记录工具（80cm靶面）</h3>

    <!-- 第一行：输入 -->
    <div style="display: flex; justify-content: center; align-items: center; gap: 40px; margin-bottom: 20px;">
        <div>
            <label>姓名：<input type="text" id="shooterName" placeholder="请输入姓名"></label><br><br>
            <label>分组：
                <select id="groupSelect">
                    <option value="3m组">3m组</option>
                    <option value="5m组">5m组</option>
                    <option value="8m组">8m组</option>
                </select>
            </label>
            <span id="arrowCount" style="margin-left:10px;color:#666;"></span>
        </div>
        <!-- 控制按钮 -->
        <div style="min-width: 150px;">
            <input type="file" id="csvFileInput" accept=".csv" onchange="importCSV()" />
            <button onclick="nextShooter()">下一位选手</button><br><br>
            <button onclick="undoLastShot()">撤回上一个点</button>
            <button onclick="exportCSV()">导出 CSV</button>
            <button onclick="clearCache()">清空缓存数据</button>
            <button onclick="clearData()">清空所有数据</button><br><br>
        </div>
    </div>

    <div style="display: flex; justify-content: center; align-items: center; gap: 40px; margin-bottom: 20px;">
        <canvas id="canvas" width="800" height="800" style="border:1px solid #ccc; cursor: crosshair;"></canvas>
    </div>

    <!-- 第二行：当前箭支数据 + 总表 -->
    <div style="display: flex; justify-content: space-around; align-items: flex-start; gap: 20px;">
        
        <!-- 当前射手数据 -->
        <div id="stats" style="flex: 1; max-width: 400px;"></div>

        <!-- 所有选手统计表 -->
        <div style="flex: 2;">
            <h4>所有选手统计汇总：</h4>
            <div id="summary"></div>
        </div>
    </div>

</body>


<script>
        // 动态适配画布
        const screenWidth = window.innerWidth || document.documentElement.clientWidth;
        const canvasSize = Math.min(screenWidth * 0.9, 800);  // 最大800，最小适配屏宽
        const canvas = document.getElementById('canvas');
        canvas.width = canvasSize;
        canvas.height = canvasSize;
        let touchTimer = null;
        const ctx = canvas.getContext('2d');
        const scale = 80 / canvas.width;  // px → cm（靶面总宽80cm）
        const center = { x: canvas.width / 2, y: canvas.height / 2 };

        const ringWidth = 4;
        const MAX_SHOTS_PER_SHOOTER = 10;
    
        let allShots = [];     // 包含所有人的记录
        let currentShots = []; // 当前射手的10箭
        let currentShooter = null;
        let chart = null;

        drawTarget();
    
        function drawTarget() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            const colors = ['#FFD700', '#FFD700', '#FF0000', '#FF0000', '#0000FF', '#0000FF', '#000000', '#000000', '#FFFFFF', '#FFFFFF'];

            // 绘制靶圈 + 环数标记
            for (let i = 10; i >= 1; i--) {
                const radius = i * ringWidth * 10;
                ctx.beginPath();
                ctx.arc(center.x, center.y, radius, 0, 2 * Math.PI);
                ctx.fillStyle = colors[i - 1];  // 正确的颜色索引：从外到内变深
                ctx.fill();
                ctx.strokeStyle = '#666';
                ctx.lineWidth = 1;
                ctx.stroke();

                // 正确的环数标记：外圈是 1，中心是 10
                const ringNumber = 11 - i;
                const midRadius = ((i + (i - 1)) / 2) * ringWidth * 10;

                ctx.fillStyle = '#000';
                ctx.font = `${canvas.width * 0.015}px Arial`;
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';

                // 添加 2cm 处的内圈线（X环）
                ctx.beginPath();
                ctx.arc(center.x, center.y, 2 / scale, 0, 2 * Math.PI);
                ctx.strokeStyle = '#333';
                ctx.lineWidth = 1;
                ctx.stroke();

                // 在右侧中心
                ctx.fillText(`${ringNumber}`, center.x + midRadius, center.y);

                // 在上方中心
                ctx.fillText(`${ringNumber}`, center.x, center.y - midRadius);
            }

            // 靶心标记
            ctx.beginPath();
            ctx.arc(center.x, center.y, 4, 0, 2 * Math.PI);
            ctx.fillStyle = '#FF0000';
            ctx.fill();
            ctx.strokeStyle = '#000';
            ctx.stroke();
            ctx.fillStyle = '#000';
            ctx.font = `${canvas.width * 0.015}px Arial`;
            ctx.fillText('靶心', center.x + 6, center.y - 6);

            // 坐标线 + 刻度
            ctx.strokeStyle = '#999';
            ctx.lineWidth = 1;

            // 横轴线
            ctx.beginPath();
            ctx.moveTo(0, center.y);
            ctx.lineTo(canvas.width, center.y);
            ctx.stroke();

            // 纵轴线
            ctx.beginPath();
            ctx.moveTo(center.x, 0);
            ctx.lineTo(center.x, canvas.height);
            ctx.stroke();

            // 刻度线和标签（每10cm）
            ctx.fillStyle = '#333';
            ctx.font = `${canvas.width * 0.012}px Arial`;
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';
            ctx.lineWidth = 1;
            const maxCm = 40;
            for (let cm = -maxCm; cm <= maxCm; cm += 10) {
                if (cm === 0) continue;
                const offset = cm / scale;

                // X轴刻度（水平线上的点）
                ctx.beginPath();
                ctx.moveTo(center.x + offset, center.y - 5);
                ctx.lineTo(center.x + offset, center.y + 5);
                ctx.stroke();
                ctx.fillText(`${cm}`, center.x + offset, center.y + 15);

                // Y轴刻度（垂直线上）
                ctx.beginPath();
                ctx.moveTo(center.x - 5, center.y - offset);
                ctx.lineTo(center.x + 5, center.y - offset);
                ctx.stroke();
                ctx.fillText(`${cm}`, center.x - 15, center.y - offset);
            }

            // 重绘当前箭痕并标出编号
            currentShots.forEach((shot, index) => {
                const px = center.x + shot.x / scale;
                const py = center.y - shot.y / scale;

                // 绘制绿色圆点
                const arrowRadiusPx = 0.4 / scale; // 0.4cm 半径，转换为像素
                ctx.fillStyle = '#00FF00';
                ctx.beginPath();
                ctx.arc(px, py, arrowRadiusPx, 0, 2 * Math.PI);
                ctx.fill();

                // 添加编号文字
                ctx.fillStyle = '#000';
                ctx.font = `${canvas.width * 0.015}px Arial`;
                ctx.textAlign = 'left';
                ctx.textBaseline = 'top';
                ctx.fillText(`${index + 1}`, px + 5, py + 5);
            });
        }
    
        canvas.addEventListener('click', (e) => {
            const nameInput = document.getElementById('shooterName');
            const groupSelect = document.getElementById('groupSelect');
            const name = nameInput.value.trim();
            const group = groupSelect.value;
    
            if (!name) {
                alert('请输入姓名');
                return;
            }
    
            if (!currentShooter) {
                currentShooter = { name, group };
                nameInput.disabled = true;
                groupSelect.disabled = true;
            }
    
            if (currentShots.length >= MAX_SHOTS_PER_SHOOTER) {
                alert('当前射手已射满10箭，请点击“下一位选手”继续。');
                return;
            }

            document.getElementById('arrowCount').textContent = `当前第 ${currentShots.length + 1} / ${MAX_SHOTS_PER_SHOOTER} 箭`;


    
            const rect = canvas.getBoundingClientRect();
            const x = e.clientX - rect.left;
            const y = e.clientY - rect.top;
            const dx = (x - center.x) * scale;
            const dy = (center.y - y) * scale;
            const distance = Math.sqrt(dx * dx + dy * dy);
            const score = getScore(distance);
    
            const shot = { x: dx, y: dy, score, name, group, timestamp: new Date().toISOString() };
            currentShots.push(shot);
            allShots.push(shot);
    
            ctx.fillStyle = '#00FF00';
            ctx.beginPath();
            ctx.arc(x, y, 0.4 / scale, 0, 2 * Math.PI);
            ctx.fill();
            
            updateStats();
            saveToLocal();
        });
    
        canvas.addEventListener('touchstart', function (e) {
            if (e.touches.length === 1) {
                const touch = e.touches[0];
                const x = touch.clientX - canvas.getBoundingClientRect().left;
                const y = touch.clientY - canvas.getBoundingClientRect().top;

                // 设置长按触发
                touchTimer = setTimeout(() => {
                    showTouchCoordinate(x, y);
                }, 600); // 长按 600ms 触发
                }
            }, { passive: false });

        canvas.addEventListener('touchend', () => {
            if (touchTimer) clearTimeout(touchTimer);
        });

        canvas.addEventListener('touchmove', () => {
            if (touchTimer) clearTimeout(touchTimer);
        });

        const getScore = (distance) => {
            const arrowRadius = 0.4; // 单位cm
            const ring = Math.floor((distance - arrowRadius + 0.0001) / ringWidth);
            const score = 10 - ring;
            return score > 0 ? score : 0;
        };
    
        function updateStats() {
            const statsDiv = document.getElementById('stats');
            if (currentShots.length === 0) {
                statsDiv.innerHTML = '';
                return;
            }
    
            const validShots = currentShots.filter(s => s.score > 0);
            let html = `<h4>当前记录者：${currentShooter.name}（${currentShooter.group}）</h4>`;
            currentShots.forEach((shot, i) => {
                html += `箭支 ${i + 1}: (${shot.x.toFixed(1)}cm, ${shot.y.toFixed(1)}cm), 环数: ${shot.score}<br>`;
            });
    
            if (validShots.length > 0) {
                const xCoords = validShots.map(s => s.x);
                const yCoords = validShots.map(s => s.y);
                const meanX = xCoords.reduce((a, b) => a + b, 0) / validShots.length;
                const meanY = yCoords.reduce((a, b) => a + b, 0) / validShots.length;
                const stdX = Math.sqrt(xCoords.reduce((a, x) => a + (x - meanX) ** 2, 0) / validShots.length);
                const stdY = Math.sqrt(yCoords.reduce((a, y) => a + (y - meanY) ** 2, 0) / validShots.length);
                const maxDist = Math.max(...validShots.map(s => Math.sqrt(s.x ** 2 + s.y ** 2)));
    
                html += `<h4>散布统计（不含脱靶）：</h4>
                    有效箭数: ${validShots.length}（总箭数: ${currentShots.length}）<br>
                    平均中心: (${meanX.toFixed(1)}cm, ${meanY.toFixed(1)}cm）<br>
                    标准差(X/Y): ${stdX.toFixed(1)}cm / ${stdY.toFixed(1)}cm<br>
                    最大距离: ${maxDist.toFixed(1)}cm`;
            } else {
                html += `<h4>散布统计：</h4>无有效箭支（全部脱靶）`;
            }
    
            statsDiv.innerHTML = html;
        }
    
        function nextShooter() {
            if (currentShots.length < MAX_SHOTS_PER_SHOOTER) {
                alert('请先完成本射手的10箭。');
                return;
            }

            currentShots = [];
            currentShooter = null;
            const nameInput = document.getElementById('shooterName');
            nameInput.value = '';
            nameInput.disabled = false;
            const groupSelect = document.getElementById('groupSelect');
            groupSelect.disabled = false;
            document.getElementById('stats').innerHTML = '';
            document.getElementById('arrowCount').textContent = '';

            drawTarget();
            updateSummary();
            saveToLocal();
        }

        function deleteShooter(name, group) {
            if (!confirm(`确认删除 ${name}（${group}）的所有记录？`)) return;
            allShots = allShots.filter(s => !(s.name === name && s.group === group));
            currentShots = [];
            currentShooter = null;
            updateSummary();
            drawTarget();
            saveToLocal();
        }

        function updateSummary() {
            const summaryDiv = document.getElementById('summary');

            // 按姓名+分组聚合
            const grouped = {};
            allShots.forEach(s => {
                const key = `${s.name}__${s.group}`;
                if (!grouped[key]) grouped[key] = [];
                grouped[key].push(s);
            });

            let html = '<table border="1" cellpadding="5" style="border-collapse: collapse;">';
            html += `<tr>
                <th>姓名</th><th>分组</th><th>总环数</th><th>平均环数</th>
                <th>散布中心X(cm)</th><th>散布中心Y(cm)</th>
                <th>标准差X</th><th>标准差Y</th><th>最大散布距离</th><th>操作</th>
            </tr>`;

            for (const key in grouped) {
                const data = grouped[key];
                const name = data[0].name;
                const group = data[0].group;
                const total = data.reduce((sum, s) => sum + s.score, 0);
                const avg = total / data.length;

                const validShots = data.filter(s => s.score > 0);
                let meanX = 0, meanY = 0, stdX = 0, stdY = 0, maxDist = 0;
                if (validShots.length > 0) {
                    const xCoords = validShots.map(s => s.x);
                    const yCoords = validShots.map(s => s.y);
                    meanX = xCoords.reduce((a, b) => a + b, 0) / validShots.length;
                    meanY = yCoords.reduce((a, b) => a + b, 0) / validShots.length;
                    stdX = Math.sqrt(xCoords.reduce((a, x) => a + (x - meanX) ** 2, 0) / validShots.length);
                    stdY = Math.sqrt(yCoords.reduce((a, y) => a + (y - meanY) ** 2, 0) / validShots.length);
                    maxDist = Math.max(...validShots.map(s => Math.sqrt(s.x ** 2 + s.y ** 2)));
                }

                html += `<tr>
                    <td>${name}</td><td>${group}</td>
                    <td>${total}</td><td>${avg.toFixed(2)}</td>
                    <td>${meanX.toFixed(2)}</td><td>${meanY.toFixed(2)}</td>
                    <td>${stdX.toFixed(2)}</td><td>${stdY.toFixed(2)}</td>
                    <td>${maxDist.toFixed(2)}</td>
                    <td><button onclick="deleteShooter('${name}', '${group}')">删除</button></td>
                </tr>`;
            }

            html += '</table>';
            summaryDiv.innerHTML = html;
        }
    
        function clearData() {
            if (!confirm('确定要清空所有数据吗？此操作无法撤销。')) return;
            allShots = [];
            currentShots = [];
            currentShooter = null;
            document.getElementById('shooterName').value = '';
            document.getElementById('shooterName').disabled = false;
            document.getElementById('groupSelect').disabled = false;
            document.getElementById('stats').innerHTML = '';
            document.getElementById('summary').innerHTML = '';
            drawTarget();
            saveToLocal();
        }

    
        function exportCSV() {
            
            if (allShots.length === 0) {
                alert('没有数据可导出。');
                return;
            }

            // 分组按 name+group
            const grouped = {};
            allShots.forEach(s => {
                const key = `${s.name}__${s.group}`;
                if (!grouped[key]) grouped[key] = [];
                grouped[key].push(s);
            });

            let csv = 'name,group,x(cm),y(cm),score,totalScore,averageScore,meanX(cm),meanY(cm),stdX(cm),stdY(cm),maxDistance(cm), timestamp\n';

            for (const key in grouped) {
                const groupData = grouped[key];
                const name = groupData[0].name;
                const group = groupData[0].group;

                // 每箭数据
                groupData.forEach(s => {
                    csv += `${s.name},${s.group},${s.x.toFixed(1)},${s.y.toFixed(1)},${s.score},${s.timestamp},,,,,,,\n`;
                });

                // 统计信息
                const totalScore = groupData.reduce((sum, s) => sum + s.score, 0);
                const averageScore = totalScore / groupData.length;

                const validShots = groupData.filter(s => s.score > 0);
                let meanX = 0, meanY = 0, stdX = 0, stdY = 0, maxDist = 0;

                if (validShots.length > 0) {
                    const xCoords = validShots.map(s => s.x);
                    const yCoords = validShots.map(s => s.y);
                    meanX = xCoords.reduce((a, b) => a + b, 0) / validShots.length;
                    meanY = yCoords.reduce((a, b) => a + b, 0) / validShots.length;
                    stdX = Math.sqrt(xCoords.reduce((a, x) => a + (x - meanX) ** 2, 0) / validShots.length);
                    stdY = Math.sqrt(yCoords.reduce((a, y) => a + (y - meanY) ** 2, 0) / validShots.length);
                    maxDist = Math.max(...validShots.map(s => Math.sqrt(s.x ** 2 + s.y ** 2)));
                }

                // 输出统计行
                csv += `${name},${group},,,,,${totalScore},${averageScore.toFixed(2)},${meanX.toFixed(2)},${meanY.toFixed(2)},${stdX.toFixed(2)},${stdY.toFixed(2)},${maxDist.toFixed(2)}\n`;
            }

            const BOM = '\uFEFF';  // UTF-8 BOM
            const blob = new Blob([BOM + csv], { type: 'text/csv;charset=utf-8;' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = 'archery_detailed_summary.csv';
            a.click();
        }

        function saveToLocal() {
            localStorage.setItem('archery_allShots', JSON.stringify(allShots));
        }

        function restoreFromLocal() {
            const data = localStorage.getItem('archery_allShots');
            if (data) {
                allShots = JSON.parse(data);
                updateSummary();

                // 尝试还原最后一个射手的数据
                if (allShots.length > 0) {
                    const last = allShots[allShots.length - 1];
                    currentShooter = { name: last.name, group: last.group };
                    document.getElementById('shooterName').value = last.name;
                    document.getElementById('groupSelect').value = last.group;
                    document.getElementById('shooterName').disabled = true;
                    document.getElementById('groupSelect').disabled = true;

                    // 恢复该射手最近10箭
                    currentShots = allShots.filter(s => s.name === last.name && s.group === last.group).slice(-10);
                    
                    // ✅ 显示箭数
                    document.getElementById('arrowCount').textContent = `当前第 ${currentShots.length} / 10 箭`;

                    drawTarget(); // 重绘箭痕
                    updateStats();
                }
            }
        }


        window.onload = function () {
            drawTarget();
            restoreFromLocal();
        };

        function clearCache() {

            if (confirm('确定清除本地缓存？刷新页面后将无数据')) {
                localStorage.removeItem('archery_allShots');
                alert('缓存已清除，请刷新页面');
            }
        }

        function importCSV() {
            const fileInput = document.getElementById('csvFileInput');
            const file = fileInput.files[0];
            if (!file) return;
            

            const reader = new FileReader();
            reader.onload = function (e) {
                const lines = e.target.result.split('\n').filter(line => line.trim() !== '');
                const newShots = [];
                const headers = lines[0].split(',');

                for (let i = 1; i < lines.length; i++) {
                    const parts = lines[i].split(',');
                    if (parts.length < 5) continue;

                    const [name, group, x, y, score, timestamp] = parts;
                    newShots.push({
                        name: name.trim(),
                        group: group.trim(),
                        x: parseFloat(x),
                        y: parseFloat(y),
                        score: parseInt(score),
                        timestamp: timestamp ? timestamp.trim() : new Date().toISOString()
                    });
                }

                allShots = newShots;
                currentShots = [];
                currentShooter = null;
                updateSummary();
                drawTarget();
                saveToLocal();
                alert('CSV 导入成功');
            };
            reader.readAsText(file);
        }

        function undoLastShot() {
            if (currentShots.length === 0) {
                alert('当前无可撤回的箭');
                return;
            }

            const lastShot = currentShots.pop();

            // 从 allShots 中移除该箭（从末尾往前找符合当前射手名和组的记录）
            for (let i = allShots.length - 1; i >= 0; i--) {
                if (allShots[i].name === lastShot.name && allShots[i].group === lastShot.group) {
                    allShots.splice(i, 1);
                    break;
                }
            }

            // 更新当前箭数显示
            document.getElementById('arrowCount').textContent = `当前第 ${currentShots.length} / 10 箭`;

            updateStats();
            drawTarget();
            saveToLocal();
        }

        function showTouchCoordinate(x, y) {
            const dx = (x - center.x) * scale;
            const dy = (center.y - y) * scale;

            const label = `(${dx.toFixed(1)}cm, ${dy.toFixed(1)}cm)`;

            // 白色背景圆
            ctx.beginPath();
            ctx.arc(x, y - 20, 28, 0, 2 * Math.PI);
            ctx.fillStyle = 'rgba(255,255,255,0.9)';
            ctx.fill();

            // 黑字标签
            ctx.fillStyle = '#000';
            ctx.font = `${canvas.width * 0.014}px Arial`;
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';
            ctx.fillText(label, x, y - 20);
        }

</script>
    
    
</body>
</html>
