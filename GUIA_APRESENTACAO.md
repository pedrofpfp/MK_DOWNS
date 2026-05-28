# GUIA COMPLETO DE APRESENTAÇÃO — DailyManager
### Sistemas Distribuídos · UNISANTOS
### Pedro de França Pereira · Miguel Nepomuceno Gil

---

> **Como usar este guia:** leia da Parte 1 até a Parte 4 em ordem. A Parte 1 explica os conceitos do zero. A Parte 2 explica cada linha do projeto. A Parte 3 cobre os tópicos avançados que o professor cobra. A Parte 4 simula as perguntas da banca.

---

# PARTE 1 — CONCEITOS FUNDAMENTAIS (do zero)

---

## 1.1 O que é um Sistema Distribuído?

Um sistema distribuído é um conjunto de computadores (ou processos) que se comunicam pela rede e parecem, para o usuário, um sistema único e coerente.

**Exemplo do cotidiano:** quando você usa o Google Docs, vários usuários editam o mesmo documento ao mesmo tempo em computadores diferentes — mas para todos parece que é um único arquivo. Isso é um sistema distribuído.

**No DailyManager:**
- O servidor roda em um computador (o seu, ou em um servidor na nuvem).
- Vários navegadores (de computadores diferentes) se conectam ao servidor.
- Quando alguém faz upload de um arquivo, **todos os outros usuários veem o arquivo aparecer na tela em tempo real** — sem recarregar a página.
- Isso é distribuído: múltiplos clientes, um servidor central, estado compartilhado.

**Arquitetura do DailyManager:** Cliente-Servidor centralizado. O servidor é o ponto único de verdade — ele decide o que existe e notifica todos os clientes.

---

## 1.2 O que é um Socket?

Um socket é um **ponto de comunicação bidirecional** entre dois processos pela rede. Pense como uma tomada: você conecta um plugue (cliente) na tomada (servidor) e a partir daí os dois podem trocar dados.

### Como o TCP funciona por baixo

O protocolo TCP (Transmission Control Protocol) garante que os dados enviados cheguem na ordem certa e sem perda. Ele opera na Camada de Transporte do modelo OSI.

```
Computador A                          Computador B
(Cliente)                             (Servidor)

    |--- SYN --------------------------->|   (quero conectar)
    |<-- SYN-ACK ------------------------|   (ok, pode)
    |--- ACK --------------------------->|   (confirmado)
    |                                    |
    |--- dados ------------------------->|   (agora troca dados)
    |<-- dados --------------------------|
    |                                    |
    |--- FIN --------------------------->|   (quero desconectar)
    |<-- FIN-ACK ------------------------|
```

Esse processo de 3 passos (SYN, SYN-ACK, ACK) é chamado de **three-way handshake** e estabelece a conexão TCP.

### Por que isso importa para o DailyManager?

Toda comunicação do DailyManager passa pelo TCP:
- O HTTP (usado nas rotas REST) roda sobre TCP.
- O WebSocket também começa como HTTP e depois "faz upgrade" para WebSocket, mas continua sobre TCP.

---

## 1.3 O que é HTTP?

HTTP (HyperText Transfer Protocol) é um protocolo de **requisição-resposta**: o cliente faz uma pergunta (requisição), o servidor responde, e a conexão pode ser encerrada.

### Anatomia de uma requisição HTTP

```
POST /upload HTTP/1.1
Host: localhost:3000
Content-Type: multipart/form-data; boundary=---boundary123

------boundary123
Content-Disposition: form-data; name="arquivo"; filename="relatorio.pdf"
Content-Type: application/pdf

[bytes do arquivo aqui]
------boundary123--
```

### Os métodos HTTP usados no DailyManager

| Método | O que significa | Onde é usado |
|--------|----------------|--------------|
| GET | "me dê dados" | Listar arquivos: `GET /arquivos` |
| POST | "receba estes dados" | Upload: `POST /upload` |
| DELETE | "apague isso" | Exclusão: `DELETE /arquivo/:id` |

### HTTP é stateless (sem estado)

Cada requisição HTTP é independente. O servidor não "lembra" do cliente entre uma requisição e outra. Por isso, quando precisamos de comunicação contínua (tempo real), usamos WebSocket.

---

## 1.4 O que é WebSocket? (e por que é diferente de HTTP)

WebSocket é um protocolo de comunicação **full-duplex** e **persistente**: após a conexão ser estabelecida, tanto o cliente quanto o servidor podem enviar mensagens a qualquer momento, sem que o outro precise pedir.

### Comparação HTTP vs WebSocket

```
HTTP (requisição-resposta):
Cliente:  "GET /arquivos"
Servidor: [responde e fecha]
Cliente:  "GET /arquivos" (de novo, 5 segundos depois)
Servidor: [responde e fecha]
→ Para saber de novidades, o cliente precisa ficar perguntando (polling)

WebSocket (conexão persistente):
Cliente:  "quero conectar via WebSocket"
Servidor: "ok, conectado"
--- conexão fica aberta ---
Servidor: "ei, chegou um arquivo novo!" (sem o cliente pedir)
Servidor: "ei, um arquivo foi deletado!" (sem o cliente pedir)
→ O servidor avisa os clientes quando algo muda (push)
```

### Como o WebSocket "começa" como HTTP

O WebSocket começa com uma requisição HTTP especial chamada **upgrade**:

```
GET / HTTP/1.1
Host: localhost:3000
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

O servidor responde com `101 Switching Protocols` e a partir daí a conexão vira WebSocket — o protocolo muda, mas a conexão TCP continua a mesma.

### WebSocket no DailyManager

```
Navegador ←──── conexão WebSocket ────→ server.js
                  (sempre aberta)

Quando alguém faz upload:
server.js ──→ "novo-arquivo" ──→ Navegador A
           ──→ "novo-arquivo" ──→ Navegador B
           ──→ "novo-arquivo" ──→ Navegador C
(todos recebem ao mesmo tempo — isso é broadcast)
```

---

## 1.5 O que é REST API?

REST (Representational State Transfer) é um estilo de arquitetura para APIs HTTP. Uma API REST organiza os recursos em URLs e usa os métodos HTTP para as operações.

No DailyManager, a API REST gerencia o recurso "arquivo":

```
GET    /arquivos        → lista todos os arquivos
POST   /upload          → cria (faz upload de) um arquivo
DELETE /arquivo/:id     → apaga o arquivo com aquele ID
GET    /uploads/:nome   → baixa (serve) o arquivo
```

O `:id` e `:nome` são **parâmetros de rota** — valores variáveis na URL. Ex: `DELETE /arquivo/42` apaga o arquivo de ID 42.

---

## 1.6 O que é o Event Loop do Node.js?

Node.js é single-threaded: roda em uma única thread de execução. Mas consegue atender múltiplos clientes ao mesmo tempo graças ao **Event Loop**.

### Como funciona

Imagine um garçom em um restaurante. Em vez de ficar parado esperando a cozinha preparar cada prato (bloqueio), ele anota o pedido e já vai atender outra mesa. Quando a cozinha avisa que o prato ficou pronto (evento), ele o entrega.

```
Event Loop (Node.js)
         │
         ▼
┌─────────────────────────────────────────┐
│  Fila de eventos                        │
│                                         │
│  [cliente A conectou]                   │
│  [cliente B enviou arquivo]             │
│  [disco terminou de gravar]             │
│  [cliente C desconectou]               │
└─────────────────────────────────────────┘
         │
         ▼
  Processa um evento por vez,
  MAS não fica bloqueado esperando I/O
```

### Por que isso é importante para sistemas distribuídos?

- Um servidor tradicional (Java com threads) cria uma thread por cliente → caro em memória.
- Node.js atende todos os clientes com uma thread → muito mais eficiente para I/O (rede, disco).
- Para o DailyManager, que pode ter dezenas de WebSocket abertos ao mesmo tempo, isso é ideal.

---

## 1.7 O que é JSON?

JSON (JavaScript Object Notation) é um formato de troca de dados legível por humanos e fácil de processar por máquinas.

```json
{
  "tipo": "novo-arquivo",
  "dados": {
    "id": 42,
    "nome": "relatorio.pdf",
    "tamanho": "128.50 KB",
    "data": "28/05/2026",
    "caminho": "1716300000000-relatorio.pdf"
  }
}
```

No DailyManager, todas as mensagens WebSocket são JSON. O servidor usa `JSON.stringify()` para transformar um objeto JavaScript em texto JSON antes de enviar, e o cliente usa `JSON.parse()` para transformar o texto de volta em objeto.

---

## 1.8 O que é SQLite?

SQLite é um banco de dados relacional que funciona como um **arquivo único** — sem servidor separado, sem instalação.

### A tabela do DailyManager

```sql
CREATE TABLE arquivos (
    id       INTEGER PRIMARY KEY AUTOINCREMENT,  -- número único automático
    nome     TEXT,   -- nome original do arquivo (ex: "relatorio.pdf")
    tamanho  TEXT,   -- tamanho formatado (ex: "128.50 KB")
    data     TEXT,   -- data do upload (ex: "28/05/2026")
    caminho  TEXT    -- nome no disco (ex: "1716300000000-relatorio.pdf")
)
```

### Por que separar "nome" de "caminho"?

- **nome:** é o nome original que o usuário vê na tela.
- **caminho:** é o nome com que o arquivo foi salvo no disco, com timestamp na frente para evitar colisões.

Exemplo: dois usuários fazem upload de `relatorio.pdf` ao mesmo tempo:
- Arquivo 1 no disco: `1716300000001-relatorio.pdf`
- Arquivo 2 no disco: `1716300000002-relatorio.pdf`
- Mas na tela os dois aparecem como `relatorio.pdf` (usando o campo `nome`)

---

## 1.9 O que é multipart/form-data?

Quando você envia um arquivo por HTTP, o conteúdo precisa ser codificado de uma forma especial: **multipart/form-data**. O arquivo é dividido em "partes" separadas por um delimitador.

```
Content-Type: multipart/form-data; boundary=----boundary

------boundary
Content-Disposition: form-data; name="arquivo"; filename="foto.jpg"
Content-Type: image/jpeg

[bytes binários da foto]
------boundary--
```

O **Multer** é o middleware que interpreta isso no Node.js, extrai o arquivo e o salva em disco automaticamente.

---

## 1.10 O que é broadcast?

Broadcast significa "enviar para todos". No DailyManager, quando um arquivo é enviado ou deletado, o servidor avisa **todos os clientes conectados via WebSocket** ao mesmo tempo.

```javascript
function enviarParaTodos(data) {
    const mensagem = JSON.stringify(data)
    clientes.forEach(cliente => {
        if (cliente.readyState === WebSocket.OPEN) {
            cliente.send(mensagem)
        }
    })
}
```

O `clientes` é um array que guarda todos os WebSocket ativos. Para cada um, o servidor envia a mensagem — se a conexão ainda estiver aberta (`readyState === WebSocket.OPEN`).

---

# PARTE 2 — O PROJETO EM DETALHES (linha por linha)

---

## 2.1 Estrutura de arquivos

```
DailyManager/
│
├── server.js          ← CÉREBRO: toda a lógica do servidor
├── package.json       ← lista de dependências (bibliotecas)
│
├── database/
│   └── database.db    ← banco de dados SQLite (arquivo binário)
│
├── uploads/           ← onde os arquivos enviados ficam salvos
│   └── 1716...-foto.jpg
│
└── web/               ← tudo que o navegador baixa
    ├── index.html     ← estrutura da página
    ├── script.js      ← lógica do cliente (JS que roda no browser)
    ├── style.css      ← visual
    └── media/
        ├── pasta-icon.png
        └── lupa-cinza.png
```

---

## 2.2 server.js — Explicado linha por linha

### Bloco 1: Importações

```javascript
const express   = require("express")     // framework HTTP
const http      = require("http")        // módulo HTTP nativo do Node
const WebSocket = require("ws")          // biblioteca WebSocket
const multer    = require("multer")      // middleware de upload de arquivos
const cors      = require("cors")        // permite acesso cross-origin
const Database  = require("better-sqlite3") // SQLite síncrono
const fs        = require("fs")          // sistema de arquivos (leitura/escrita)
```

**Por que `http` separado do `express`?**
O Express cria um servidor HTTP internamente, mas para o WebSocket precisamos do objeto `server` "cru" do módulo `http`. Por isso criamos o servidor na mão e passamos o Express como "handler".

### Bloco 2: Criação do servidor

```javascript
const app    = express()                    // cria a aplicação Express
const server = http.createServer(app)       // cria servidor HTTP usando Express
const wss    = new WebSocket.Server({ server }) // WebSocket usa o MESMO servidor
```

**Ponto crítico:** o WebSocket Server é criado com `{ server }` — isso faz com que **HTTP e WebSocket compartilhem a mesma porta 3000**. Quando o navegador faz upgrade para WebSocket, o mesmo servidor que atende HTTP passa a atender WebSocket.

### Bloco 3: Middlewares

```javascript
app.use(cors())              // permite requisições de qualquer origem
app.use(express.json())      // interpreta body como JSON automaticamente
app.use(express.static("web"))      // serve os arquivos de /web/ como estáticos
app.use("/uploads", express.static("uploads")) // serve os arquivos enviados
```

**O que é um middleware?** É uma função que roda no meio do caminho, antes do handler da rota. O Express processa middlewares em ordem. Se eu faço `GET /index.html`, o `express.static("web")` encontra o arquivo e responde antes de chegar em qualquer rota que eu defini.

### Bloco 4: Banco de dados

```javascript
const db = new Database("./database/database.db")

db.prepare(`
    CREATE TABLE IF NOT EXISTS arquivos(
        id      INTEGER PRIMARY KEY AUTOINCREMENT,
        nome    TEXT,
        tamanho TEXT,
        data    TEXT,
        caminho TEXT
    )
`).run()
```

`CREATE TABLE IF NOT EXISTS` significa: "crie a tabela se ela ainda não existir". Então na primeira vez que o servidor roda, a tabela é criada. Nas vezes seguintes, o banco já tem os dados e nada é apagado.

### Bloco 5: Configuração do Multer (armazenamento de arquivos)

```javascript
const storage = multer.diskStorage({
    destination: function(req, file, cb) {
        cb(null, "uploads/")           // salvar na pasta uploads/
    },
    filename: function(req, file, cb) {
        cb(null, Date.now() + "-" + file.originalname)
        // ex: 1716300000000-relatorio.pdf
    }
})

const upload = multer({ storage })
```

`Date.now()` retorna o timestamp atual em milissegundos (ex: `1716300000000`). Isso garante que cada arquivo tenha um nome único no disco, mesmo que dois usuários enviem arquivos com o mesmo nome ao mesmo tempo.

### Bloco 6: Gerenciamento de conexões WebSocket

```javascript
let clientes = []   // array que guarda todos os WebSocket ativos

wss.on("connection", (ws) => {
    console.log("Cliente conectado")
    clientes.push(ws)              // adiciona o novo cliente

    ws.on("close", () => {
        clientes = clientes.filter(c => c !== ws)  // remove ao desconectar
    })
})
```

**Por que filtrar em vez de `splice`?**
`filter` cria um novo array sem o cliente desconectado. É mais seguro porque não modifica o array enquanto pode estar sendo iterado em um broadcast.

### Bloco 7: Função de broadcast

```javascript
function enviarParaTodos(data) {
    const mensagem = JSON.stringify(data)   // converte objeto para texto JSON

    clientes.forEach(cliente => {
        if (cliente.readyState === WebSocket.OPEN) {
            cliente.send(mensagem)
        }
    })
}
```

`readyState === WebSocket.OPEN` verifica se a conexão ainda está ativa antes de tentar enviar. Sem essa verificação, tentar enviar para um socket fechado causaria um erro.

### Bloco 8: Rota de upload (POST /upload)

```javascript
app.post("/upload", upload.single("arquivo"), (req, res) => {
    const arquivo = req.file   // o Multer já salvou o arquivo em disco
                               // e colocou as infos aqui

    // formata o tamanho em KB com 2 casas decimais
    const tamanho = (arquivo.size / 1024).toFixed(2) + " KB"

    // formata a data como DD/MM/YYYY
    const hoje = new Date()
    const data =
        hoje.getDate().toString().padStart(2, "0") + "/" +
        (hoje.getMonth() + 1).toString().padStart(2, "0") + "/" +
        hoje.getFullYear()

    // insere no banco de dados
    const result = db.prepare(`
        INSERT INTO arquivos(nome, tamanho, data, caminho)
        VALUES (?, ?, ?, ?)
    `).run(
        arquivo.originalname,   // nome original (ex: relatorio.pdf)
        tamanho,                // ex: 128.50 KB
        data,                   // ex: 28/05/2026
        arquivo.filename        // nome no disco (ex: 1716...-relatorio.pdf)
    )

    // notifica todos os clientes via WebSocket
    enviarParaTodos({
        tipo: "novo-arquivo",
        dados: {
            id: result.lastInsertRowid,  // ID gerado pelo SQLite
            nome: arquivo.originalname,
            tamanho,
            data,
            caminho: arquivo.filename
        }
    })

    res.send("ok")   // responde ao cliente que fez o upload
})
```

**Fluxo completo:** Multer intercepta a requisição → salva o arquivo em disco → `req.file` fica disponível → registramos no banco → fazemos broadcast → respondemos `"ok"`.

**Por que `?` nos prepared statements?**
Os `?` são placeholders que o `better-sqlite3` substitui pelos valores de forma segura, prevenindo **SQL Injection** (ataque onde o usuário colocaria SQL malicioso no nome do arquivo).

### Bloco 9: Rota de listagem (GET /arquivos)

```javascript
app.get("/arquivos", (req, res) => {
    const arquivos = db.prepare("SELECT * FROM arquivos").all()

    // filtra arquivos que existem fisicamente no disco
    const arquivosValidos = arquivos.filter(arquivo => {
        return fs.existsSync("./uploads/" + arquivo.caminho)
    })

    res.json(arquivosValidos)
})
```

**Por que verificar se o arquivo existe no disco?**
Pode acontecer de alguém deletar manualmente um arquivo da pasta `uploads/` sem passar pela API. Sem essa verificação, o banco mostraria arquivos que não existem mais. O `fs.existsSync()` verifica se o arquivo está fisicamente lá antes de incluir na resposta.

### Bloco 10: Rota de exclusão (DELETE /arquivo/:id)

```javascript
app.delete("/arquivo/:id", (req, res) => {
    const id = req.params.id   // pega o :id da URL

    // busca o arquivo no banco
    const arquivo = db.prepare("SELECT * FROM arquivos WHERE id=?").get(id)

    if (!arquivo) return res.sendStatus(404)   // não encontrado

    // remove fisicamente do disco
    fs.unlinkSync("./uploads/" + arquivo.caminho)

    // remove do banco de dados
    db.prepare("DELETE FROM arquivos WHERE id=?").run(id)

    // notifica todos via WebSocket
    enviarParaTodos({
        tipo: "arquivo-deletado",
        dados: id
    })

    res.send("ok")
})
```

**Ordem das operações:**
1. Busca no banco (para saber o caminho no disco)
2. Remove do disco (`fs.unlinkSync`)
3. Remove do banco
4. Broadcast para todos os clientes

Se invertêssemos 2 e 3, e o servidor caísse entre eles, teríamos um registro no banco apontando para um arquivo que não existe mais. A ordem atual garante que o pior caso seja um arquivo no disco sem registro no banco (que é detectado pelo `existsSync` na listagem).

### Bloco 11: Inicia o servidor

```javascript
server.listen(3000, () => {
    console.log("Servidor rodando em http://localhost:3000")
})
```

O `server.listen(3000)` faz o servidor começar a aceitar conexões na porta 3000. A porta é um número que identifica qual processo na máquina deve receber o tráfego (como um ramal de telefone).

---

## 2.3 script.js — O Frontend em detalhes

### Bloco 1: Conecta ao WebSocket

```javascript
const wsProtocol = location.protocol === "https:" ? "wss:" : "ws:"
const socket = new WebSocket(`${wsProtocol}//${location.host}`)
```

**Por que detectar o protocolo?**
- Em `http://localhost:3000` → usa `ws://localhost:3000` (WebSocket sem criptografia)
- Em `https://abc.ngrok-free.app` → usa `wss://abc.ngrok-free.app` (WebSocket com criptografia)

Se usássemos sempre `ws://` em HTTPS, o navegador bloquearia a conexão por segurança (mixed content). Esse código resolve automaticamente.

`location.host` pega o hostname e porta da URL atual — então funciona em qualquer ambiente (localhost, ngrok, IP local) sem precisar hardcodar o endereço.

### Bloco 2: Recebe eventos do servidor

```javascript
socket.onmessage = (event) => {
    const msg = JSON.parse(event.data)   // converte JSON de texto para objeto

    if (msg.tipo === "novo-arquivo") {
        criarItem(msg.dados)   // adiciona a linha na tabela
    }

    if (msg.tipo === "arquivo-deletado") {
        const item = document.querySelector(`[data-id="${msg.dados}"]`)
        if (item) item.remove()   // remove o elemento do DOM
    }
}
```

Quando o servidor faz broadcast, essa função é chamada em todos os navegadores conectados. O `msg.tipo` determina o que fazer:
- `"novo-arquivo"` → cria uma linha nova na tabela
- `"arquivo-deletado"` → encontra a linha pelo `data-id` e remove

### Bloco 3: Cria um item na lista

```javascript
function criarItem(arquivo) {
    const item = document.createElement("a")   // elemento clicável (link)

    item.classList.add("linha-arquivo")
    item.dataset.id = arquivo.id    // guarda o ID no elemento (para poder deletar depois)
    item.href = "/uploads/" + arquivo.caminho

    item.innerHTML = `
        <div class="col-nome">${arquivo.nome}</div>
        <div class="col-tamanho">${arquivo.tamanho}</div>
        <div class="col-data">${arquivo.data}</div>
        <div class="col-acoes">
            <button class="botao-baixar">⬇</button>
            <button class="botao-excluir">🗑</button>
        </div>
    `

    // botão de exclusão
    const botaoExcluir = item.querySelector(".botao-excluir")
    botaoExcluir.addEventListener("click", async (e) => {
        e.preventDefault()   // evita navegar para o href do link
        await fetch("/arquivo/" + arquivo.id, { method: "DELETE" })
        // NÃO remove do DOM aqui — o broadcast faz isso para todos
    })

    // botão de download
    const botaoBaixar = item.querySelector(".botao-baixar")
    botaoBaixar.addEventListener("click", (e) => {
        e.preventDefault()
        const link = document.createElement("a")
        link.href = "/uploads/" + arquivo.caminho
        link.download = arquivo.nome   // força download com nome original
        link.click()
    })

    listaArquivos.appendChild(item)
}
```

**Por que o botão de exclusão não remove do DOM diretamente?**
Porque o servidor vai fazer broadcast de `arquivo-deletado` para TODOS os clientes, incluindo o que clicou no botão. O `socket.onmessage` vai remover o elemento. Assim, todos os usuários (não só o que clicou) veem o arquivo sumir.

### Bloco 4: Carrega arquivos ao abrir a página

```javascript
async function carregarArquivos() {
    const res = await fetch("/arquivos")     // GET /arquivos
    const arquivos = await res.json()        // converte resposta para array

    arquivos.forEach(arquivo => {
        criarItem(arquivo)    // cria uma linha para cada arquivo
    })
}

carregarArquivos()   // executa imediatamente ao carregar a página
```

Isso garante que ao abrir a página você já vê todos os arquivos existentes, mesmo que tenham sido enviados em sessões anteriores.

### Bloco 5: Upload

```javascript
botaoArquivo.addEventListener("click", () => {
    seletorArquivo.click()   // abre o seletor de arquivo do sistema operacional
})

seletorArquivo.addEventListener("change", async function() {
    const arquivo = this.files[0]    // pega o primeiro arquivo selecionado
    if (!arquivo) return

    const formData = new FormData()
    formData.append("arquivo", arquivo)    // campo "arquivo" = o arquivo selecionado

    await fetch("/upload", {
        method: "POST",
        body: formData    // o fetch detecta FormData e define Content-Type automaticamente
    })
    // NÃO adiciona ao DOM aqui — o broadcast faz isso
})
```

### Bloco 6: Busca local

```javascript
barraBuscar.addEventListener("input", function() {
    const busca = barraBuscar.value.toLowerCase()
    const arquivos = document.querySelectorAll(".linha-arquivo")

    arquivos.forEach(item => {
        const nome = item.innerText.toLowerCase()

        if (nome.includes(busca)) {
            item.style.display = "grid"   // mostra
        } else {
            item.style.display = "none"   // esconde
        }
    })
})
```

A busca é **local** — filtra os elementos já presentes no DOM sem fazer nenhuma requisição ao servidor. Isso é rápido e não gera tráfego de rede.

---

## 2.4 index.html — Estrutura da página

```html
<nav class="navbar">
    <!-- logo + título -->
    <div class="container-logo">
        <img src="media/pasta-icon.png" class="logo">
        <h1>DailyManager</h1>
    </div>
    <!-- barra de busca centralizada -->
    <div class="container-buscar">
        <img src="media/lupa-cinza.png">
        <input type="text" id="barraBuscar" placeholder="Buscar item....">
    </div>
</nav>

<div class="gerenciador">
    <!-- coluna lateral: botão de upload -->
    <div class="coluna-arquivos">
        <button class="botao-adicionar">+ Novo arquivo</button>
        <input type="file" id="seletorArquivo" style="display:none">
        <!-- o input file fica escondido; o botão o dispara via JS -->
    </div>

    <!-- painel principal: tabela de arquivos -->
    <div class="painel-arquivos">
        <div class="cabecalho-arquivos">
            <div class="col-nome">Nome</div>
            <div class="col-tamanho">Tamanho</div>
            <div class="col-data">Data de Modificação</div>
        </div>
        <div class="lista-arquivos"></div>
        <!-- itens são inseridos aqui dinamicamente pelo script.js -->
    </div>
</div>

<!-- carrega o script (deve ser no final do body) -->
<script src="script.js"></script>
```

---

# PARTE 3 — TÓPICOS AVANÇADOS (o professor vai perguntar)

---

## 3.1 Criptografia — TLS/SSL

### Estado atual do projeto

O DailyManager atualmente comunica em **texto plano** — sem criptografia. Isso significa que alguém com acesso à rede (por exemplo, na mesma rede Wi-Fi) pode usar ferramentas como **Wireshark** para capturar os pacotes TCP e ver:
- Os nomes dos arquivos enviados
- O conteúdo dos arquivos
- Os metadados das operações

### Onde a criptografia deve ser aplicada

A criptografia protege a **camada de transporte** — a comunicação entre o navegador e o servidor. Ela atua em dois canais:

1. **HTTP → HTTPS:** as requisições REST (upload, delete, listar)
2. **WS → WSS (WebSocket Secure):** as notificações em tempo real

### Como implementar TLS

```javascript
// Em vez de:
const http   = require("http")
const server = http.createServer(app)

// Usaria:
const https  = require("https")
const fs     = require("fs")

const server = https.createServer({
    key:  fs.readFileSync("server-key.pem"),   // chave privada RSA
    cert: fs.readFileSync("server-cert.pem")   // certificado X.509
}, app)

// O WebSocket não precisa mudar — usa o mesmo server
const wss = new WebSocket.Server({ server })
```

### O que muda no front-end?

**Nada.** O `script.js` já detecta automaticamente:
```javascript
const wsProtocol = location.protocol === "https:" ? "wss:" : "ws:"
```
Com HTTPS habilitado, o código automaticamente usará `wss://` — que é WebSocket sobre TLS.

### Como o TLS funciona (simplificado)

```
Cliente                           Servidor
  │── ClientHello ─────────────────→│  (versão TLS, algoritmos suportados)
  │←── ServerHello + Certificado ───│  (escolhe algoritmo, manda certificado)
  │── verifica certificado ─────────│  (o navegador confia na CA?)
  │── ClientKeyExchange ───────────→│  (troca de chaves para gerar chave simétrica)
  │←───────── Finished ─────────────│
  │═══════════════════════════════════  (a partir daqui, tudo cifrado com AES-256)
  │── GET /arquivos (cifrado) ──────→│
  │←── resposta (cifrada) ───────────│
```

O resultado é que mesmo que alguém capture os pacotes TCP, só verá bytes aleatórios — sem a chave privada do servidor, é impossível decifrar.

### Certificado gratuito: Let's Encrypt

```bash
# Instalar certbot
sudo apt install certbot

# Gerar certificado para um domínio
sudo certbot certonly --standalone -d meudominio.com.br
```

---

## 3.2 DNS — Sistema de Nomes de Domínio

### O que é DNS

DNS é o sistema que traduz nomes legíveis (ex: `google.com`) para endereços IP (ex: `142.250.80.14`). Sem DNS, você precisaria memorizar IPs para acessar qualquer site.

### Como funciona a resolução DNS

```
Você digita: dailymanager.com.br
      │
      ▼
Computador pergunta ao Resolver DNS local (geralmente o roteador)
      │
      ▼
Resolver pergunta ao Root DNS ("quem cuida de .br?")
      │
      ▼
Root DNS responde: "o servidor X cuida de .br"
      │
      ▼
Resolver pergunta ao servidor de .br ("quem cuida de .com.br?")
      │
      ▼
...até chegar ao servidor autoritativo do domínio
      │
      ▼
Responde: "dailymanager.com.br = IP 203.0.113.10"
      │
      ▼
Seu computador conecta ao IP 203.0.113.10 porta 80/443
```

### Estado atual do projeto

O DailyManager usa:
- `localhost:3000` — acesso local (não passa pelo DNS)
- Via ngrok: `https://abc123.ngrok-free.app` — o ngrok já gerencia o DNS do subdomínio

### O ngrok e DNS

O ngrok cria um túnel criptografado entre a internet e o seu servidor local:

```
Usuário externo
      │
      ▼
abc123.ngrok-free.app (DNS do ngrok aponta para servidores do ngrok)
      │
      ▼
Servidores do ngrok (na nuvem)
      │  (túnel TLS)
      ▼
ngrok no seu computador
      │
      ▼
localhost:3000 (seu servidor Node.js)
```

### Para produção real, o que seria necessário

1. Registrar um domínio (ex: no Registro.br — custo ~R$40/ano para `.com.br`)
2. No painel do registrador, criar um registro DNS tipo **A**:
   ```
   dailymanager.com.br  →  203.0.113.10  (IP do servidor)
   ```
3. Se o IP mudar (IP dinâmico residencial), usar **DDNS** (Dynamic DNS):
   - Serviço como DuckDNS detecta a mudança de IP e atualiza o registro automaticamente

### Relevância para o WebSocket

O `script.js` usa `location.host` para a URL do WebSocket:
```javascript
const socket = new WebSocket(`${wsProtocol}//${location.host}`)
```
Então `location.host` seria `dailymanager.com.br` — e o DNS resolve para o IP antes da conexão TCP. **Nenhuma mudança de código necessária.**

---

## 3.3 Armazenamento, Consistência e Replicação

### Modelo de armazenamento do DailyManager

O sistema separa dois tipos de dados:

```
┌─────────────────────────────────────────────────────────┐
│  METADADOS (o que é o arquivo)                          │
│  SQLite: database/database.db                           │
│  Tabela: arquivos                                       │
│  id | nome | tamanho | data | caminho                   │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  DADOS BRUTOS (o conteúdo do arquivo)                   │
│  Sistema de arquivos: uploads/                          │
│  Arquivo: 1716300000000-relatorio.pdf                   │
└─────────────────────────────────────────────────────────┘
```

### Consistência

Consistência significa que o banco e o disco sempre concordam sobre o que existe.

**Problema:** e se o processo morrer no meio da operação de exclusão?

```javascript
fs.unlinkSync("./uploads/" + arquivo.caminho)   // passo 1: remove do disco
// → servidor cai aqui ←
db.prepare("DELETE FROM arquivos WHERE id=?").run(id)  // passo 2: nunca executado
```

Resultado: o banco tem um registro, mas o arquivo não existe no disco (registro órfão).

**Solução adotada:** filtrar na listagem:
```javascript
const arquivosValidos = arquivos.filter(arquivo => {
    return fs.existsSync("./uploads/" + arquivo.caminho)
})
```

Arquivos órfãos são simplesmente ignorados na resposta. Uma solução mais robusta seria uma transação no banco que só confirma após a exclusão do disco.

### Replicação (discussão teórica)

**O que é replicação?** Manter cópias dos dados em múltiplos servidores para que, se um cair, os outros continuem funcionando.

**SQLite não tem replicação nativa.** Para um sistema de produção real, existem duas abordagens:

1. **Migrar para PostgreSQL** com replicação streaming:
   - Um servidor "primário" recebe escritas
   - Servidores "réplica" recebem cópias e servem leituras
   - Se o primário cair, uma réplica assume

2. **Litestream** (mantém SQLite mas adiciona replicação):
   - Captura o WAL (Write-Ahead Log) do SQLite em tempo real
   - Replica para S3, GCS ou outro armazenamento externo
   - Em caso de falha, restaura de lá

**Para os arquivos físicos:**
- Substituir `uploads/` (disco local) por **MinIO** (S3 auto-hospedado) ou **AWS S3**
- Múltiplos servidores acessam o mesmo bucket — não dependem do disco local

---

## 3.4 Ataques de Comunicação — DDoS e Interceptação

### DDoS (Distributed Denial of Service)

Um atacante sobrecarrega o servidor com tantas requisições ou conexões que ele para de responder para usuários legítimos.

**Tipos mais comuns:**
- **Volume:** enviar gigas de tráfego (satura a banda de rede)
- **Connection flood:** abrir milhares de conexões TCP simultâneas (esgota os file descriptors do processo)
- **Slowloris:** abrir conexões HTTP e enviar os dados muito devagar, mantendo o servidor esperando

**Vulnerabilidade do DailyManager:**
```javascript
wss.on("connection", (ws) => {
    clientes.push(ws)   // sem limite de conexões!
})
```
Um atacante poderia abrir 10.000 conexões WebSocket e esgotar a memória do processo.

**Mitigações possíveis:**

```javascript
// 1. Limite global de conexões
server.maxConnections = 100   // no máximo 100 conexões simultâneas

// 2. Rate limiting por IP (usando express-rate-limit)
const rateLimit = require("express-rate-limit")
app.use(rateLimit({
    windowMs: 60 * 1000,   // janela de 1 minuto
    max: 20                // máximo 20 requisições por IP por minuto
}))

// 3. Em produção: Cloudflare na frente do servidor
// O Cloudflare absorve o tráfego DDoS antes de chegar ao Node.js
```

### Interceptação de Pacotes (Sniffing)

Sem TLS, qualquer dispositivo na mesma rede pode capturar os pacotes usando Wireshark ou tcpdump e ver o conteúdo dos arquivos enviados.

```
Rede Wi-Fi pública:
   [Seu computador] ──── HTTP sem criptografia ────→ [Servidor]
         │
   [Atacante na mesma rede Wi-Fi]
         │
   Wireshark captura os pacotes e vê tudo
```

**Com TLS:**
```
   [Seu computador] ════ HTTPS (AES-256-GCM) ═════→ [Servidor]
         │
   [Atacante na mesma rede Wi-Fi]
         │
   Wireshark captura bytes aleatórios — inúteis sem a chave privada
```

### Man-in-the-Middle (MITM)

Sem autenticação do servidor, um atacante pode se colocar entre o cliente e o servidor, interceptando e modificando o tráfego.

```
Normal:
Cliente ──→ Servidor

MITM:
Cliente ──→ Atacante ──→ Servidor
         ←──          ←──
(o atacante vê e pode modificar tudo)
```

O TLS previne isso com certificados X.509: o navegador verifica se o certificado do servidor foi emitido por uma Autoridade Certificadora (CA) confiável. Se o atacante tentar se passar pelo servidor, não terá o certificado válido.

---

# PARTE 4 — SIMULAÇÃO DAS PERGUNTAS DO PROFESSOR

---

## Pergunta 1: "O que é um sistema distribuído e como o DailyManager se enquadra nessa definição?"

**Resposta completa:**

Um sistema distribuído é um conjunto de processos autônomos que se comunicam pela rede e, para o usuário, parecem um sistema único coerente. As características fundamentais são: múltiplos nós de computação, comunicação por rede, e compartilhamento de estado.

O DailyManager se enquadra nessa definição de três formas:

**1. Múltiplos clientes simultâneos:** vários navegadores, rodando em computadores diferentes, acessam o sistema ao mesmo tempo. Cada navegador é um nó cliente do sistema distribuído.

**2. Estado compartilhado:** a lista de arquivos é compartilhada entre todos. Quando o cliente A faz upload de `relatorio.pdf`, o cliente B vê o arquivo aparecer na tela — ambos enxergam o mesmo estado do sistema.

**3. Comunicação pela rede:** clientes e servidor trocam mensagens pelo protocolo WebSocket (para tempo real) e HTTP REST (para operações). Essa comunicação pode acontecer em rede local ou pela internet via ngrok.

A arquitetura é **Cliente-Servidor centralizado**: o servidor é o ponto único de autoridade — ele recebe todas as operações, decide o que é válido, e notifica todos os clientes sobre mudanças. Isso simplifica a consistência (não há conflitos entre servidores) mas cria um ponto único de falha.

---

## Pergunta 2: "Como funciona o WebSocket no projeto? Por que não usaram HTTP puro?"

**Resposta completa:**

HTTP é um protocolo **stateless** de requisição-resposta: o cliente pergunta, o servidor responde, a conexão é encerrada. Para saber se algo mudou, o cliente teria que ficar fazendo requisições periodicamente (polling):

```javascript
// Com HTTP puro, precisaria de polling:
setInterval(async () => {
    const res = await fetch("/arquivos")
    const arquivos = await res.json()
    atualizarTela(arquivos)
}, 2000)  // perguntar a cada 2 segundos
```

Problemas do polling: gera tráfego desnecessário, tem latência (se alguém enviar um arquivo, os outros podem demorar até 2 segundos para ver), e escala mal com muitos clientes.

**WebSocket resolve isso:** após a conexão ser estabelecida, ela fica aberta permanentemente. O servidor pode enviar mensagens para o cliente a qualquer momento, sem o cliente precisar pedir.

No `server.js`:
```javascript
// Quando um arquivo é uploaded, o servidor avisa TODOS imediatamente:
enviarParaTodos({
    tipo: "novo-arquivo",
    dados: { id: ..., nome: ..., tamanho: ..., data: ..., caminho: ... }
})
```

No `script.js`, o cliente escuta:
```javascript
socket.onmessage = (event) => {
    const msg = JSON.parse(event.data)
    if (msg.tipo === "novo-arquivo") criarItem(msg.dados)
}
```

A latência cai de segundos para milissegundos, e o servidor só envia dados quando há novidade — muito mais eficiente.

---

## Pergunta 3: "Explique como o upload de arquivo funciona do início ao fim."

**Resposta completa:**

O fluxo completo de um upload tem 6 etapas:

**Etapa 1 — Seleção pelo usuário** (`script.js`):
```javascript
botaoArquivo.addEventListener("click", () => {
    seletorArquivo.click()  // abre o seletor de arquivos do SO
})
```
O usuário clica em `+ Novo arquivo` → o seletor de arquivos abre.

**Etapa 2 — Envio via HTTP** (`script.js`):
```javascript
const formData = new FormData()
formData.append("arquivo", arquivo)
await fetch("/upload", { method: "POST", body: formData })
```
O arquivo é empacotado em `multipart/form-data` e enviado via `POST /upload`.

**Etapa 3 — Multer recebe e salva em disco** (`server.js`):
```javascript
const storage = multer.diskStorage({
    filename: (req, file, cb) => cb(null, Date.now() + "-" + file.originalname)
})
```
O Multer intercepta a requisição, salva o arquivo em `uploads/1716...-nome.ext` e preenche `req.file` com as informações.

**Etapa 4 — SQLite registra os metadados** (`server.js`):
```javascript
const result = db.prepare(
    "INSERT INTO arquivos(nome, tamanho, data, caminho) VALUES (?,?,?,?)"
).run(arquivo.originalname, tamanho, data, arquivo.filename)
```
Os metadados são persistidos no banco. `result.lastInsertRowid` dá o ID gerado.

**Etapa 5 — Broadcast via WebSocket** (`server.js`):
```javascript
enviarParaTodos({
    tipo: "novo-arquivo",
    dados: { id: result.lastInsertRowid, nome: ..., tamanho: ..., ... }
})
```
Todos os clientes conectados recebem a mensagem instantaneamente.

**Etapa 6 — DOM atualizado em todos os clientes** (`script.js`):
```javascript
socket.onmessage = (event) => {
    const msg = JSON.parse(event.data)
    if (msg.tipo === "novo-arquivo") criarItem(msg.dados)
}
```
Cada cliente cria uma nova linha na tabela com os dados recebidos.

---

## Pergunta 4: "Por que o arquivo salvo em disco tem um nome diferente do nome original?"

**Resposta completa:**

O Multer salva o arquivo com um nome prefixado por timestamp:

```javascript
filename: function(req, file, cb) {
    cb(null, Date.now() + "-" + file.originalname)
    // ex: 1716300000000-relatorio.pdf
}
```

`Date.now()` retorna o número de milissegundos desde 1º de janeiro de 1970 (Unix timestamp). Esse número é único em cada momento.

**Por que isso é necessário?** Imagine dois usuários enviando `relatorio.pdf` ao mesmo tempo:
- Sem prefixo: ambos tentariam criar `uploads/relatorio.pdf` → o segundo sobrescreveria o primeiro → perda de arquivo!
- Com prefixo: `uploads/1716300000001-relatorio.pdf` e `uploads/1716300000002-relatorio.pdf` → ambos coexistem sem conflito

O banco armazena o **nome original** (`nome`) separadamente do **nome em disco** (`caminho`), então o usuário sempre vê o nome original na interface:

```sql
| id | nome          | caminho                         |
|----|---------------|---------------------------------|
|  1 | relatorio.pdf | 1716300000001-relatorio.pdf     |
|  2 | relatorio.pdf | 1716300000002-relatorio.pdf     |
```

Dois arquivos com o mesmo nome para o usuário, mas fisicamente diferentes no disco.

---

## Pergunta 5: "Como o sistema garante que a lista de arquivos está sempre correta, mesmo se um arquivo for deletado manualmente do disco?"

**Resposta completa:**

Existe uma possível inconsistência entre banco e disco: se alguém deletar manualmente um arquivo da pasta `uploads/` sem usar a API (ou se o servidor travar durante a exclusão), o banco ficaria com um registro que aponta para um arquivo inexistente.

A solução está na rota `GET /arquivos`:

```javascript
app.get("/arquivos", (req, res) => {
    const arquivos = db.prepare("SELECT * FROM arquivos").all()

    const arquivosValidos = arquivos.filter(arquivo => {
        return fs.existsSync("./uploads/" + arquivo.caminho)
    })

    res.json(arquivosValidos)
})
```

`fs.existsSync(caminho)` retorna `true` se o arquivo físico existe, `false` caso contrário. O `.filter()` mantém apenas os registros cujo arquivo existe no disco.

Resultado: arquivos órfãos (no banco mas não no disco) são simplesmente ignorados na resposta — o usuário nunca os vê. Eles ficam no banco mas são filtrados em toda listagem.

Uma solução mais robusta seria limpar periodicamente esses registros órfãos do banco, mas para o escopo do projeto, o filtro em tempo real é suficiente e simples.

---

## Pergunta 6: "O que acontece se dois usuários clicarem em 'deletar' no mesmo arquivo ao mesmo tempo?"

**Resposta completa:**

Este é um cenário de **condição de corrida** (race condition). Vejamos o que acontece:

**Usuário A clica em deletar arquivo ID 5**
**Usuário B clica em deletar arquivo ID 5** (praticamente ao mesmo tempo)

```javascript
// Requisição A chega primeiro:
const arquivo = db.prepare("SELECT * FROM arquivos WHERE id=?").get(5)
// → encontra o arquivo

fs.unlinkSync("./uploads/" + arquivo.caminho)
// → arquivo removido do disco

db.prepare("DELETE FROM arquivos WHERE id=?").run(5)
// → removido do banco

// Requisição B chega logo depois:
const arquivo = db.prepare("SELECT * FROM arquivos WHERE id=?").get(5)
// → retorna undefined (já foi deletado!)

if (!arquivo) return res.sendStatus(404)
// → responde 404, não tenta deletar novamente
```

A verificação `if (!arquivo) return res.sendStatus(404)` protege contra o segundo delete. No pior caso, o usuário B recebe um 404, mas nada quebra.

No front-end, o broadcast do primeiro delete já remove o elemento do DOM de todos os clientes, então o botão "deletar" do usuário B nem deveria mais existir na tela quando ele clicar.

**Limitação:** como o `better-sqlite3` é síncrono no Node.js single-thread, na prática as requisições são processadas uma de cada vez pelo Event Loop, então a chance de colisão real é muito baixa.

---

## Pergunta 7: "Como funciona a detecção automática de ws:// vs wss://? Por que isso é importante para o ngrok?"

**Resposta completa:**

No `script.js`:
```javascript
const wsProtocol = location.protocol === "https:" ? "wss:" : "ws:"
const socket = new WebSocket(`${wsProtocol}//${location.host}`)
```

`location.protocol` é uma propriedade do navegador que contém o protocolo da URL atual: `"http:"` ou `"https:"`.

**Por que isso importa para o ngrok:**

Quando você usa ngrok, ele cria uma URL HTTPS:
```
https://abc123.ngrok-free.app
```

O navegador está em HTTPS. Regra de segurança dos browsers: uma página carregada via HTTPS **não pode** fazer conexões não-seguras (chamado "mixed content"). Se tentássemos usar `ws://` em uma página HTTPS, o navegador bloquearia a conexão.

**Sem a detecção automática (errado):**
```javascript
const socket = new WebSocket("ws://abc123.ngrok-free.app")
// ❌ Bloqueado! Página é HTTPS, conexão WS seria HTTP (inseguro)
```

**Com a detecção automática (correto):**
```javascript
// location.protocol === "https:" → wsProtocol = "wss:"
const socket = new WebSocket("wss://abc123.ngrok-free.app")
// ✅ WebSocket Secure — compatível com HTTPS
```

O ngrok funciona como proxy TLS: a conexão entre o usuário e os servidores do ngrok é segura (WSS), e o ngrok internamente faz a ponte para o `ws://localhost:3000` do servidor local. O código não precisa saber disso — basta usar o mesmo hostname da URL da página.

---

## Pergunta 8: "O que é SQL Injection e como o projeto previne esse ataque?"

**Resposta completa:**

SQL Injection é quando um atacante insere código SQL malicioso em campos de entrada para manipular o banco de dados.

**Exemplo de ataque (sem proteção):**
```javascript
// Código vulnerável:
const nome = req.body.nome  // usuário coloca: "'; DROP TABLE arquivos; --"
db.exec(`INSERT INTO arquivos(nome) VALUES ('${nome}')`)
// SQL resultante:
// INSERT INTO arquivos(nome) VALUES (''); DROP TABLE arquivos; --')
// → executa o DROP TABLE e apaga todos os dados!
```

**Como o DailyManager previne:**

O `better-sqlite3` usa **prepared statements** com placeholders `?`:

```javascript
db.prepare(
    "INSERT INTO arquivos(nome, tamanho, data, caminho) VALUES (?, ?, ?, ?)"
).run(arquivo.originalname, tamanho, data, arquivo.filename)
```

Os `?` são placeholders. O banco de dados trata os valores como **dados**, nunca como código SQL. Mesmo que o nome do arquivo seja `'; DROP TABLE arquivos; --`, ele será inserido como texto literal na coluna `nome` — sem executar SQL malicioso.

O mesmo vale para as outras queries:
```javascript
db.prepare("SELECT * FROM arquivos WHERE id=?").get(id)
db.prepare("DELETE FROM arquivos WHERE id=?").run(id)
```

O parâmetro `:id` vem da URL e pode ser manipulado por um atacante. Com prepared statements, tentativas de injection são inofensivas.

---

## Pergunta 9: "Por que Node.js foi escolhido? Como o Event Loop permite múltiplos clientes simultâneos sem múltiplas threads?"

**Resposta completa:**

Node.js foi escolhido por três razões principais:

**1. I/O não-bloqueante:** operações de disco e rede não bloqueiam o thread principal. Enquanto o Multer salva um arquivo em disco, o servidor pode atender outra requisição.

**2. Mesmo linguagem no front e back:** o `script.js` (navegador) e o `server.js` (servidor) são ambos JavaScript — facilita o desenvolvimento e o compartilhamento de lógica.

**3. Ecossistema rico:** Express, ws, Multer, better-sqlite3 — todas as bibliotecas necessárias estão disponíveis como pacotes npm maduros.

**Como o Event Loop funciona:**

```
Thread único do Node.js
         │
         ▼
┌────────────────────────────────────┐
│           Event Loop               │
│                                    │
│  1. Processa evento da fila        │
│  2. Se o evento requer I/O:        │
│     → delega ao sistema operacional│
│     → NÃO espera — vai para o próx│
│  3. Quando o I/O termina:          │
│     → callback entra na fila       │
│  4. Repete                         │
└────────────────────────────────────┘
```

**Exemplo prático com 3 clientes:**

```
t=0ms: Cliente A faz upload de 50MB
       → Multer começa a salvar no disco (I/O assíncrono)
       → Event Loop NÃO espera, vai para próximo evento

t=1ms: Cliente B abre conexão WebSocket
       → registrado no array clientes[]
       → Event Loop continua

t=2ms: Cliente C lista arquivos
       → SQLite executa SELECT (síncrono, rápido)
       → retorna resposta

t=500ms: Disco terminou de salvar o arquivo do Cliente A
         → callback é chamado
         → SQLite registra, broadcast enviado
```

Um servidor Java tradicional criaria 3 threads (um por cliente), cada uma bloqueada esperando I/O. Node.js usa uma thread com callbacks — muito mais eficiente para I/O-bound como servidores de arquivos.

---

## Pergunta 10: "O que precisa ser feito para colocar o DailyManager em produção real? Quais são as limitações atuais?"

**Resposta completa:**

O DailyManager funciona bem para demonstração acadêmica, mas tem várias limitações para uso em produção:

**Limitação 1: Sem criptografia**
- Atual: HTTP + WS (texto plano)
- Produção: HTTPS + WSS com certificado TLS (ex: Let's Encrypt gratuito)
- Impacto: sem TLS, senhas, arquivos e dados ficam expostos na rede

**Limitação 2: Sem autenticação**
- Atual: qualquer pessoa com o link pode fazer upload, download e excluir qualquer arquivo
- Produção: sistema de login com JWT (JSON Web Tokens) ou sessões
- Impacto: sem autenticação, qualquer pessoa na internet pode deletar todos os arquivos

**Limitação 3: Sem limite de tamanho ou tipo de arquivo**
- Atual: o Multer aceita qualquer arquivo de qualquer tamanho
- Produção: limitar tamanho (ex: 100MB) e tipos (ex: bloquear .exe)
- Impacto: um usuário pode encher o disco do servidor com arquivos gigantes

**Limitação 4: SQLite em servidor único**
- Atual: SQLite em arquivo local, um servidor
- Produção: PostgreSQL com replicação streaming para alta disponibilidade
- Impacto: se o servidor cair, o serviço fica indisponível

**Limitação 5: Arquivos em disco local**
- Atual: `uploads/` no disco do servidor
- Produção: armazenamento de objetos (S3/MinIO) para escalabilidade horizontal
- Impacto: não é possível ter múltiplas instâncias do servidor compartilhando os mesmos arquivos

**Limitação 6: Sem proteção contra DDoS**
- Atual: sem rate limiting — qualquer IP pode abrir milhares de conexões
- Produção: `express-rate-limit`, `server.maxConnections`, Cloudflare
- Impacto: um atacante pode derrubar o servidor com um script simples

**O que já está bem para produção:**
- Prevenção de SQL Injection com prepared statements ✅
- Consistência banco-disco com `fs.existsSync` ✅
- Prevenção de colisão de nomes com timestamp ✅
- Detecção automática ws/wss para compatibilidade HTTPS ✅
- Remoção de clientes desconectados do broadcast ✅

---

# RESUMO RÁPIDO — Para consulta durante a apresentação

```
TECNOLOGIAS:
  Node.js     → runtime JavaScript, Event Loop, I/O não-bloqueante
  Express 5   → servidor HTTP, roteamento, middlewares
  ws          → WebSocket de baixo nível (sem socket.io)
  Multer      → recebe multipart/form-data, salva arquivo em disco
  better-sqlite3 → banco SQLite síncrono, prepared statements

ARQUITETURA:
  Navegadores ──HTTP REST──→ server.js ──→ SQLite (metadados)
             ←──WS Broadcast──            ──→ uploads/ (arquivos)

FLUXO UPLOAD:
  click → POST /upload → Multer salva disco → SQLite insere
  → enviarParaTodos(novo-arquivo) → criarItem() em todos os clientes

FLUXO DELETE:
  click → DELETE /arquivo/:id → unlinkSync → SQLite delete
  → enviarParaTodos(arquivo-deletado) → item.remove() em todos

TEMPO REAL:
  ws.onmessage = JSON.parse → tipo → ação no DOM

CONSISTÊNCIA:
  GET /arquivos filtra com fs.existsSync() antes de retornar

CRIPTOGRAFIA:
  Atual: sem criptografia (HTTP/WS texto plano)
  Solução: TLS → HTTPS (módulo https) + WSS (auto via location.protocol)

DNS:
  Atual: localhost ou ngrok
  Produção: registro A no DNS → domínio.com.br = IP do servidor

SEGURANÇA:
  SQL Injection: prevenido por prepared statements (?)
  DDoS: mitigação com rate limiting + server.maxConnections + Cloudflare
  Sniffing: mitigado com TLS

REPLICAÇÃO:
  SQLite: sem suporte nativo → migrar para PostgreSQL ou usar Litestream
  Arquivos: disco local → migrar para MinIO/S3
```

---

*Boa apresentação! Você entende o projeto — só fale com calma e apontando para o código.*
