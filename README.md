<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<title>Калькулятор ЗИП (ГОСТ 27.507-2015)</title>
<style>
    body { font-family: sans-serif; margin: 20px; background: #f9f9f9; }
    .container { max-width: 900px; background: white; padding: 20px; border-radius: 8px; box-shadow: 0 0 10px rgba(0,0,0,0.1); }
    table { width: 100%; border-collapse: collapse; margin-top: 15px; }
    td, th { border: 1px solid #ddd; padding: 8px; text-align: center; }
    input[type="number"], select { width: 100%; padding: 4px; }
    button { background: #007bff; color: white; border: none; padding: 10px 20px; cursor: pointer; margin-top: 10px; }
    .result-box { background: #d1ecf1; padding: 15px; border-radius: 5px; margin-top: 20px; display: none; }
</style>
</head>
<body>
<div class="container">
    <h1>Оценка показателей достаточности ЗИП (ГОСТ 27.507-2015)</h1>
    
    <label><strong>Тип комплекта:</strong></label>
    <select id="kitType">
        <option value="O">ЗИП-О (Одиночный)</option>
        <option value="G">ЗИП-Г (Групповой)</option>
    </select>
    <br><br>
    
    <table id="partsTable">
        <thead>
            <tr>
                <th>Наименование</th>
                <th>n (в изделии)</th>
                <th>Lambda (1/ч)</th>
                <th>Стоимость (руб)</th>
                <th>Стратегия</th>
                <th>t_п (ч)</th>
                <th>t_д (ч)</th>
                <th>N (изделий)</th>
            </tr>
        </thead>
        <tbody>
            <!-- Строка 1 -->
            <tr>
                <td><input type="text" value="Блок питания"></td>
                <td><input type="number" value="2" step="1"></td>
                <td><input type="number" value="0.0001" step="0.0000001"></td>
                <td><input type="number" value="15000" step="1"></td>
                <td><select><option value="3">3 (НП)</option><option value="1">1 (П)</option></select></td>
                <td><input type="number" value="8760" step="1"></td>
                <td><input type="number" value="150" step="1"></td>
                <td><input type="number" value="10" step="1"></td>
            </tr>
            <!-- Строка 2 -->
            <tr>
                <td><input type="text" value="Резистор"></td>
                <td><input type="number" value="50" step="1"></td>
                <td><input type="number" value="0.00001" step="0.0000001"></td>
                <td><input type="number" value="50" step="1"></td>
                <td><select><option value="1">1 (П)</option><option value="3">3 (НП)</option></select></td>
                <td><input type="number" value="8760" step="1"></td>
                <td><input type="number" value="0" step="1"></td>
                <td><input type="number" value="10" step="1"></td>
            </tr>
        </tbody>
    </table>
    <button onclick="addRow()">+ Добавить позицию</button>
    <br><br>
    <label>Требуемый K_г (для прямой задачи): <input type="number" id="targetKG" value="0.98" step="0.001"></label>
    <br><br>
    <button onclick="calculateResults()">Рассчитать показатели (оценка при h=0)</button>

    <div id="results" class="result-box">
        <h3>Результаты расчета:</h3>
        <p id="outText"></p>
    </div>
</div>

<script>
function addRow() {
    const table = document.querySelector('#partsTable tbody');
    const newRow = table.insertRow();
    const cellsHTML = `
        <td><input type="text"></td>
        <td><input type="number" value="1"></td>
        <td><input type="number" value="0.0001" step="0.0000001"></td>
        <td><input type="number" value="100"></td>
        <td><select><option value="3">3 (НП)</option><option value="1">1 (П)</option><option value="2">2 (ПЭД)</option></select></td>
        <td><input type="number" value="8760"></td>
        <td><input type="number" value="150"></td>
        <td><input type="number" value="1"></td>`;
    newRow.innerHTML = cellsHTML;
}

function calculateResults() {
    const rows = document.querySelectorAll('#partsTable tbody tr');
    let totalOmega = 0;
    let totalQ = 0;
    let kitType = document.getElementById('kitType').value;
    
    rows.forEach(row => {
        const inputs = row.querySelectorAll('input');
        const selects = row.querySelectorAll('select');
        
        const n = parseFloat(inputs[1].value);
        const lambda = parseFloat(inputs[2].value);
        const cost = parseFloat(inputs[3].value);
        const strategy = parseInt(selects[0].value);
        const tp = parseFloat(inputs[5].value);
        const td = parseFloat(inputs[6].value);
        const N = parseFloat(inputs[7].value);
        
        // Интенсивность спроса Omega
        const omega = lambda * N * n;
        
        // Среднее число заявок rho за период (соответствует формулам 9.1 - 9.6 ГОСТ)
        let rho = 0;
        if (strategy === 3) { // НП
            rho = omega * td;
        } else { // П или ПЭД (упрощенно по времени периода)
            rho = omega * tp; 
        }
        
        // Показатель недостаточности Q при текущем запасе h=0
        let q_h = 0;
        if (rho > 0) {
            q_h = 1 - Math.exp(-rho);
        }
        
        totalOmega += omega;
        totalQ += q_h;
    });
    
    const T_zad = totalQ / totalOmega;
    const K_g = 1 - (T_zad / 8760); // Оценка коэффициента готовности через годовой фонд времени
    
    const resBox = document.getElementById('results');
    const outP = document.getElementById('outText');
    resBox.style.display = 'block';
    
    outP.innerHTML = `<b>Суммарная интенсивность спроса ($\Omega$):</b> ${totalOmega.toExponential(4)} 1/ч<br>
                      <b>Суммарный показатель недостаточности ($Q_{\Sigma}$):</b> ${totalQ.toPrecision(4)}<br>
                      <b>Среднее время задержки ($T_{зад}$):</b> ${T_zad.toFixed(4)} ч<br>
                      <b>Коэффициент готовности ЗИП ($K_{г}$):</b> ${K_g.toPrecision(4)}<br>
                      <hr>
                      <i>Примечание: Это оценка для пустого склада (h=0). Согласно методике 10.1.2 ГОСТ, для получения оптимального состава необходимо итерационно добавлять по 1 шт. детали того типа, где отношение $(Q_h(i) - Q_h(i+1)) / c_i$ максимально.</i>`;
}
</script>
</body>
</html>
