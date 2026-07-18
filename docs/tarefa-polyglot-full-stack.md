# 🏠 TAREFA PARA CASA: Polyglot Full-Stack com Docker Compose

⏱️ **Tempo estimado:** 2–3h | 🎯 **Foco:** Orquestração multi-serviço, CORS, volumes, redes internas

## 📐 Arquitetura

```
[Frontend:8080] ──CORS──> 🌐 Node API :3000 ──> 📦 PostgreSQL :5432
                        🌐 Python API :8000 ──> 📦 PostgreSQL :5432 (compartilhado)
```

- **Frontend:** HTML/JS estático servido via Nginx
- **Backend 1 (Node.js):** CRUD de `todos`
- **Backend 2 (Python):** Leitura de estatísticas do mesmo banco
- **Banco:** PostgreSQL com volume nomeado

> ✅ **Objetivo Docker:** Containerizar 4 serviços, resolver comunicação entre portas diferentes (CORS), garantir persistência e usar `docker compose` para orquestrar tudo com um comando.
> 

---

## 📁 Estrutura Obrigatória

```
polyglot-compose/
├── docker-compose.yml
├── frontend/
│   ├── Dockerfile
│   └── index.html
├── node-api/
│   ├── Dockerfile
│   ├── package.json
│   └── index.js
├── python-svc/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── main.py
└── README.md
```

---

## 💻 Código Pronto para Copiar

### 🌐 `frontend/index.html` (Vanilla JS)

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <title>Polyglot Docker App</title>
  <style>
    body { font-family: system-ui, sans-serif; max-width: 800px; margin: 2rem auto; padding: 0 1rem; background: #f8f9fa; }
    .card { background: #fff; border: 1px solid #ddd; padding: 1.5rem; margin: 1rem 0; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.05); }
    input, button { padding: 0.5rem; margin: 0.3rem 0; }
    button { cursor: pointer; background: #0d6efd; color: white; border: none; border-radius: 4px; }
    ul { list-style: none; padding: 0; }
    li { padding: 0.4rem 0; border-bottom: 1px solid #eee; }
  </style>
</head>
<body>
  <h1>🐳 App Polyglot (Node + Python + Docker)</h1>

  <div class="card">
    <h2>📝 Gerenciar Todos (Node.js API)</h2>
    <input id="todoInput" placeholder="Nova tarefa...">
    <button onclick="addTodo()">Adicionar</button>
    <ul id="todoList"></ul>
  </div>

  <div class="card">
    <h2>📊 Estatísticas (Python FastAPI)</h2>
    <button onclick="loadStats()">Atualizar Stats</button>
    <p id="statsOutput">Clique para carregar...</p>
  </div>

  <script>
    const NODE_API = 'http://localhost:3000';
    const PYTHON_API = 'http://localhost:8000';

    async function loadTodos() {
      const res = await fetch(`${NODE_API}/todos`);
      const todos = await res.json();
      document.getElementById('todoList').innerHTML = todos.map(t => `<li>${t.title}</li>`).join('');
    }

    async function addTodo() {
      const title = document.getElementById('todoInput').value;
      await fetch(`${NODE_API}/todos`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ title })
      });
      document.getElementById('todoInput').value = '';
      loadTodos();
    }

    async function loadStats() {
      const res = await fetch(`${PYTHON_API}/stats`);
      const stats = await res.json();
      document.getElementById('statsOutput').innerText =
        `Total de todos: ${stats.total_todos} | Tamanho médio do título: ${stats.avg_title_length.toFixed(1)} caracteres`;
    }

    window.onload = loadTodos;
  </script>
</body>
</html>
```

### 🟢 `node-api/package.json` *(adicionado `cors`)*

```json
{
  "name": "node-api",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": { "express": "^4.18.2", "pg": "^8.11.3", "cors": "^2.8.5" }
}
```

### 🟢 `node-api/index.js` *(adicionado `cors()`)*

```jsx
const express = require('express');
const cors = require('cors');
const { Pool } = require('pg');
const app = express();

app.use(cors()); // ✅ Permite requisições do frontend
app.use(express.json());

const pool = new Pool({
  host: process.env.DB_HOST,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
});

app.get('/todos', async (_, res) => {
  const { rows } = await pool.query('SELECT * FROM todos ORDER BY id DESC');
  res.json(rows);
});

app.post('/todos', async (req, res) => {
  const { title } = req.body;
  const { rows } = await pool.query('INSERT INTO todos (title) VALUES ($1) RETURNING *', [title]);
  res.status(201).json(rows[0]);
});

pool.query(`CREATE TABLE IF NOT EXISTS todos (id SERIAL PRIMARY KEY, title TEXT, created_at TIMESTAMP DEFAULT NOW())`)
  .then(() => app.listen(3000, () => console.log('✅ Node API rodando na porta 3000')));
```

### 🐍 `python-svc/main.py` *(adicionado CORS)*

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import psycopg2, os

app = FastAPI()

# ✅ Permite requisições do frontend
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

def get_db():
    return psycopg2.connect(
        host=os.getenv("DB_HOST"),
        database=os.getenv("DB_NAME"),
        user=os.getenv("DB_USER"),
        password=os.getenv("DB_PASSWORD")
    )

@app.get("/stats")
def stats():
    conn = get_db()
    cur = conn.cursor()
    cur.execute("SELECT COUNT(*), AVG(LENGTH(title)) FROM todos")
    row = cur.fetchone()
    return {"total_todos": row[0], "avg_title_length": float(row[1]) if row[1] else 0}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### 🐍 `python-svc/Dockerfile` & `requirements.txt`

`requirements.txt`

```
fastapi==0.104.1
uvicorn==0.24.0
psycopg2-binary==2.9.9
```

`Dockerfile`

```docker
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0"]
```

### 🌐 `frontend/Dockerfile` (Nginx estático)

```docker
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

## 🛠️ `docker-compose.yml` (Esqueleto para completar)

> ⚠️ **Desafio:** Preencha os `_________` usando os conceitos do Encontro 1.
> 

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: polyglot
      POSTGRES_PASSWORD: secret
    volumes:
      - _________:/var/lib/postgresql/data

  node-api:
    build: ./node-api
    ports: ["3000:3000"]
    environment:
      DB_HOST: db
      DB_USER: postgres
      DB_PASSWORD: secret
      DB_NAME: polyglot
    depends_on:
      - db

  python-svc:
    build: ./python-svc
    ports: ["_________:8000"]
    environment:
      DB_HOST: _________
      DB_USER: postgres
      DB_PASSWORD: secret
      DB_NAME: polyglot
    depends_on:
      - db

  frontend:
    build: ./frontend
    ports: ["8080:80"]
    depends_on:
      - node-api
      - python-svc

volumes:
  _________
```

---

## ✅ Instruções de Execução & Teste

1. Na raiz: `docker compose up -d --build`
2. Aguarde ~10s. Abra `http://localhost:8080`
3. Adicione um todo → clique em "Atualizar Stats" → veja a contagem aumentar.
4. Teste persistência: `docker compose down && docker compose up -d` → os dados devem permanecer.
5. Verifique saúde: `docker compose ps` → todos devem estar `Up`.

---

## 📋 Critérios de Aceite

- [ ]  `docker compose up -d` sobe 4 serviços sem erros
- [ ]  Frontend responde em `localhost:8080`
- [ ]  Node API em `3000`, Python em `8000`
- [ ]  Ambos acessam o **mesmo** banco e tabela
- [ ]  Dados persistem após `down`/`up`
- [ ]  CORS configurado corretamente (sem erros no console do navegador)
- [ ]  `README.md` com instruções de uso e arquitetura

---

## 🩺 Tabela de Debug (Entregue com a tarefa)

| Sintoma | Causa Provável | Como investigar |
| --- | --- | --- |
| `Blocked by CORS policy` | Backend sem CORS ou frontend usando IP errado | `docker compose logs node-api` → adicione `app.use(cors())` |
| `fetch failed` ou `Network Error` | Frontend tentando `http://localhost` dentro do container Nginx | Nginx roda no container, mas o JS roda no **navegador**. `localhost` aponta para a máquina host. ✅ Correto! |
| Python `ModuleNotFoundError` | `requirements.txt` não copiado ou `pip install` falhou | `docker compose build python-svc` → veja output do build |
| DB `connection refused` | `DB_HOST` errado ou Postgres ainda iniciando | `docker compose logs db \| grep "ready"` |
| Frontend 404 | `COPY` no Dockerfile apontando para arquivo errado | `docker compose exec frontend ls /usr/share/nginx/html` |

---