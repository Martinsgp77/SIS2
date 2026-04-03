from flask import Flask, request, jsonify, render_template_string
import sqlite3
from datetime import datetime

app = Flask(__name__)
DB = "petri.db"

# ------------------ DATABASE ------------------
def init_db():
    conn = sqlite3.connect(DB)
    c = conn.cursor()

    c.execute('''CREATE TABLE IF NOT EXISTS vendedores (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        nome TEXT
    )''')

    c.execute('''CREATE TABLE IF NOT EXISTS vendas (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        vendedor_id INTEGER,
        valor REAL,
        produto TEXT,
        data TEXT
    )''')

    conn.commit()
    conn.close()

init_db()

# ------------------ FRONTEND ------------------
HTML = """
<!DOCTYPE html>
<html>
<head>
    <title>Petri Dashboard</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
</head>
<body style="font-family: Arial; text-align:center;">

<h1>📊 Dashboard PETRI</h1>
<h2 id="faturamento">Carregando...</h2>

<h3>🏆 Ranking</h3>
<ul id="ranking"></ul>

<hr>

<h3>💰 Registrar Venda</h3>
<input id="vendedor" placeholder="ID Vendedor"><br><br>
<input id="valor" placeholder="Valor"><br><br>

<select id="produto">
<option>Box primeira experiência</option>
<option>Assinatura</option>
<option>Kit da semana</option>
<option>Charuto do dia</option>
<option>Outros</option>
</select><br><br>

<button onclick="registrarVenda()">Registrar</button>

<script>
function carregarDashboard(){
    fetch('/dashboard')
    .then(r => r.json())
    .then(data => {
        document.getElementById('faturamento').innerText =
        'Faturamento Hoje: R$ ' + data.faturamento_dia;

        let ranking = '';
        data.ranking.forEach(r => {
            ranking += `<li>${r[0]} - R$ ${r[1]}</li>`;
        });

        document.getElementById('ranking').innerHTML = ranking;
    });
}

function registrarVenda(){
    fetch('/venda', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({
            vendedor_id: document.getElementById('vendedor').value,
            valor: document.getElementById('valor').value,
            produto: document.getElementById('produto').value
        })
    }).then(() => {
        alert('Venda registrada!');
        carregarDashboard();
    });
}

setInterval(carregarDashboard, 3000);
carregarDashboard();
</script>

</body>
</html>
"""

@app.route('/')
def home():
    return render_template_string(HTML)

# ------------------ BACKEND ------------------

@app.route('/vendedor', methods=['POST'])
def criar_vendedor():
    data = request.json
    conn = sqlite3.connect(DB)
    c = conn.cursor()
    c.execute("INSERT INTO vendedores (nome) VALUES (?)", (data['nome'],))
    conn.commit()
    conn.close()
    return jsonify({"msg": "Vendedor criado"})

@app.route('/venda', methods=['POST'])
def registrar_venda():
    data = request.json
    conn = sqlite3.connect(DB)
    c = conn.cursor()
    c.execute(
        "INSERT INTO vendas (vendedor_id, valor, produto, data) VALUES (?, ?, ?, ?)",
        (data['vendedor_id'], data['valor'], data['produto'], datetime.now())
    )
    conn.commit()
    conn.close()
    return jsonify({"msg": "Venda registrada"})

@app.route('/dashboard', methods=['GET'])
def dashboard():
    conn = sqlite3.connect(DB)
    c = conn.cursor()

    hoje = datetime.now().strftime('%Y-%m-%d')

    c.execute("SELECT SUM(valor) FROM vendas WHERE date(data)=?", (hoje,))
    total = c.fetchone()[0] or 0

    c.execute("""
        SELECT vendedor_id, SUM(valor)
        FROM vendas
        GROUP BY vendedor_id
        ORDER BY SUM(valor) DESC
    """)
    ranking = c.fetchall()

    conn.close()

    return jsonify({
        "faturamento_dia": total,
        "ranking": ranking
    })

# ------------------ RUN ------------------
if __name__ == '__main__':
    app.run(debug=True)