




# PERGUNTAS DA BANCA — DailyManager
### 15 perguntas difíceis com respostas detalhadas

---

# BLOCO 1 — "Aponte no código onde isso acontece"

---

## Pergunta 1
**"O sistema garante que dois uploads simultâneos do mesmo arquivo não se sobrescrevem. Onde exatamente no código isso é tratado e qual é o mecanismo usado?"**

### Resposta

Em `server.js`, dentro da configuração do Multer:

```javascript
const storage = multer.diskStorage({
    destination: function(req, file, cb){
        cb(null, "uploads/")
    },
    filename: function(req, file, cb){
        cb(null, Date.now() + "-" + file.originalname)  // ← AQUI
    }
})
```

O mecanismo é o prefixo `Date.now()` — timestamp em milissegundos desde 01/01/1970. Mesmo que dois clientes enviem `foto.jpg` no exato mesmo instante, cada upload recebe um timestamp diferente (ex: `1716300000001-foto.jpg` e `1716300000002-foto.jpg`), garantindo nomes únicos no disco.

Porém há uma sutileza importante: o `Date.now()` tem resolução de **1 milissegundo**. Em teoria, dois uploads que chegam no mesmo milissegundo poderiam gerar o mesmo nome. Uma solução mais robusta usaria `crypto.randomUUID()` no lugar do timestamp. Para o escopo do projeto, o timestamp é suficiente.

O banco armazena o nome original separado do caminho em disco exatamente por causa disso:
```javascript
db.prepare(`INSERT INTO arquivos(nome, tamanho, data, caminho) VALUES (?,?,?,?)`)
.run(
    arquivo.originalname,  // "foto.jpg" — o que o usuário vê
    tamanho,
    data,
    arquivo.filename       // "1716...-foto.jpg" — o que existe no disco
)
```

---

## Pergunta 2
**"Onde no código o sistema detecta que um cliente WebSocket desconectou e o que é feito com essa informação? O que aconteceria se isso não fosse tratado?"**

### Resposta

Em `server.js`:

```javascript
wss.on("connection", (ws) => {
    clientes.push(ws)               // adiciona ao array

    ws.on("close", () => {          // ← AQUI: evento de desconexão
        clientes = clientes.filter(c => c !== ws)  // remove do array
    })
})
```

O evento `"close"` é emitido pelo `ws` quando a conexão WebSocket é encerrada — seja por fechamento normal do navegador, queda de rede ou timeout.

**O que aconteceria sem esse tratamento:**

A função `enviarParaTodos` itera sobre o array `clientes` e chama `.send()` em cada um:

```javascript
function enviarParaTodos(data) {
    const mensagem = JSON.stringify(data)
    clientes.forEach(cliente => {
        if (cliente.readyState === WebSocket.OPEN) {
            cliente.send(mensagem)   // ← lançaria erro em socket morto
        }
    })
}
```

Sem remover os clientes desconectados, o array cresceria indefinidamente. A cada upload ou delete, o servidor tentaria enviar para todos os sockets já fechados. O `readyState === WebSocket.OPEN` previne o envio imediato, mas o array continuaria crescendo — **memory leak** progressivo que eventualmente derrubaria o servidor.

---

## Pergunta 3
**"Onde no código o sistema evita retornar arquivos que existem no banco mas não existem mais no disco? Por que esse cenário pode ocorrer?"**

### Resposta

Em `server.js`, na rota `GET /arquivos`:

```javascript
app.get("/arquivos", (req, res) => {
    const arquivos = db.prepare("SELECT * FROM arquivos").all()

    const arquivosValidos = arquivos.filter(arquivo => {
        return fs.existsSync("./uploads/" + arquivo.caminho)  // ← AQUI
    })

    res.json(arquivosValidos)
})
```

O `fs.existsSync()` verifica a existência física do arquivo no disco antes de incluí-lo na resposta.

**Cenários em que isso pode ocorrer:**

1. **Crash entre operações:** a rota de delete executa `fs.unlinkSync` (remove do disco) e depois `db.prepare("DELETE").run()` (remove do banco). Se o processo morrer entre essas duas linhas, o banco terá um registro mas o arquivo não estará no disco.

2. **Deleção manual:** alguém acessa o servidor via SSH e deleta arquivos diretamente da pasta `uploads/` sem passar pela API.

3. **Limpeza de disco automática:** algum script de manutenção que apague arquivos antigos sem consultar o banco.

Sem o `existsSync`, o usuário veria arquivos listados na interface que, ao clicar para baixar, retornariam erro 404.

---

## Pergunta 4
**"Onde no código o projeto resolve o problema de compatibilidade entre HTTP local e HTTPS do ngrok para o WebSocket? Explique a lógica condicional."**

### Resposta

Em `web/script.js`, primeira linha:

```javascript
const wsProtocol = location.protocol === "https:" ? "wss:" : "ws:"
const socket = new WebSocket(`${wsProtocol}//${location.host}`)
```

**A lógica:**

`location.protocol` é uma propriedade nativa do browser que retorna o protocolo da URL atual, incluindo os dois pontos: `"http:"` ou `"https:"`.

- Se a página foi carregada via `http://localhost:3000` → `location.protocol === "https:"` é `false` → usa `"ws:"`
- Se a página foi carregada via `https://abc.ngrok-free.app` → `location.protocol === "https:"` é `true` → usa `"wss:"`

**Por que isso é obrigatório com ngrok:**

Browsers implementam a política de **Mixed Content**: uma página servida via HTTPS não pode iniciar conexões não-seguras (HTTP ou WS). Se tentasse `ws://` em contexto HTTPS, o browser bloquearia a conexão com erro:

```
Mixed Content: The page was loaded over HTTPS, but attempted to 
connect to the insecure WebSocket endpoint 'ws://...'. 
This request has been blocked.
```

`location.host` pega o hostname e porta atuais (`localhost:3000` ou `abc.ngrok-free.app`), então a URL do WebSocket aponta sempre para o mesmo servidor que serviu a página — sem hardcode de endereço.

---

## Pergunta 5
**"Onde no código está a prevenção contra SQL Injection e como ela funciona tecnicamente?"**

### Resposta

Em três pontos do `server.js`, todos usando **prepared statements** com placeholders `?`:

```javascript
// Inserção (upload):
db.prepare(`INSERT INTO arquivos(nome,tamanho,data,caminho) VALUES (?,?,?,?)`)
  .run(arquivo.originalname, tamanho, data, arquivo.filename)  // ← AQUI

// Busca (delete):
db.prepare("SELECT * FROM arquivos WHERE id=?").get(id)  // ← AQUI

// Remoção (delete):
db.prepare("DELETE FROM arquivos WHERE id=?").run(id)    // ← AQUI
```

**Como funciona tecnicamente:**

Um prepared statement é compilado pelo banco de dados em duas etapas separadas:

1. **Etapa de compilação:** o banco recebe `"SELECT * FROM arquivos WHERE id=?"` e compila a estrutura da query. Os `?` são marcadores — o banco sabe exatamente onde os valores virão.

2. **Etapa de execução:** os valores reais (`id`, `nome`, etc.) são passados como **dados**, nunca como texto SQL a ser interpretado.

**Ataque sem prepared statement:**
```javascript
// Vulnerável:
const query = `SELECT * FROM arquivos WHERE id=${req.params.id}`
// Se id = "1 OR 1=1" → retorna TODOS os registros
// Se id = "1; DROP TABLE arquivos; --" → apaga o banco
```

**Com prepared statement:**
```javascript
// Seguro:
db.prepare("SELECT * FROM arquivos WHERE id=?").get("1 OR 1=1")
// O banco trata "1 OR 1=1" como valor literal da coluna id
// Nenhum registro com id="1 OR 1=1" existe → retorna undefined
```

O `better-sqlite3` garante que os valores nunca são concatenados na string SQL — são passados em um canal separado diretamente para o engine do SQLite.

---

# BLOCO 2 — "O que este trecho de código faz?"

---

## Pergunta 6
**Analise este trecho e explique o que acontece, em que ordem, e qual seria o impacto de inverter as linhas 2 e 3:**

```javascript
fs.unlinkSync("./uploads/" + arquivo.caminho)           // linha 1
db.prepare("DELETE FROM arquivos WHERE id=?").run(id)   // linha 2

enviarParaTodos({                                        // linha 3
    tipo: "arquivo-deletado",
    dados: id
})
```

### Resposta

**O que faz em ordem:**

**Linha 1:** `fs.unlinkSync` remove o arquivo fisicamente do disco de forma **síncrona** (bloqueia o Event Loop até terminar). O `"./uploads/" + arquivo.caminho` monta o path completo, ex: `./uploads/1716300000000-relatorio.pdf`. Após essa linha, o arquivo não existe mais no sistema operacional.

**Linha 2:** o banco de dados SQLite remove o registro correspondente. Após essa linha, o arquivo não existe mais nos metadados.

**Linha 3:** `enviarParaTodos` serializa `{ tipo: "arquivo-deletado", dados: id }` para JSON e envia via WebSocket para todos os clientes conectados. Cada cliente recebe e remove a linha correspondente do DOM.

**Impacto de inverter linhas 2 e 3 (broadcast antes de deletar do banco):**

Cenário: o servidor envia o broadcast, e no intervalo de microssegundos antes de executar o DELETE do banco, um cliente faz `GET /arquivos`. O servidor retornaria o arquivo como válido (ainda no banco), e o `existsSync` passaria se o arquivo ainda estiver no disco. O estado ficaria temporariamente inconsistente.

**Impacto de inverter linhas 1 e 2 (deletar do banco antes do disco):**

Se o servidor crashar após o DELETE do banco mas antes do `unlinkSync`, o arquivo permanece no disco sem registro no banco. Ele ocupa espaço mas nunca será listado ou acessível pela API — arquivo zumbi. A ordem atual (disco primeiro) é pior no sentido de que pode gerar registro órfão, mas o `existsSync` na listagem cobre esse caso. Arquivo sem registro no banco é mais difícil de detectar e limpar.

---

## Pergunta 7
**O que este trecho faz? Por que `clientes` é reatribuído com `filter` em vez de modificar o array com `splice`?**

```javascript
let clientes = []

wss.on("connection", (ws) => {
    clientes.push(ws)

    ws.on("close", () => {
        clientes = clientes.filter(c => c !== ws)
    })
})
```

### Resposta

**O que faz:**

- `clientes.push(ws)`: a cada nova conexão WebSocket, o socket é adicionado ao array. `ws` é o objeto que representa a conexão daquele cliente específico — com ele é possível enviar mensagens, verificar o estado da conexão, e detectar eventos.

- `clientes.filter(c => c !== ws)`: quando aquele cliente desconecta (evento `"close"`), cria um novo array contendo todos os clientes **exceto** o que fechou. O resultado é reatribuído a `clientes`.

**Por que `filter` em vez de `splice`:**

`splice` modifica o array original **in place**. Se `enviarParaTodos` estiver iterando sobre `clientes` com `forEach` exatamente no momento em que `splice` remove um elemento, o índice interno do `forEach` fica desincronizado — elementos podem ser pulados ou processados duas vezes. É uma condição de corrida sutil.

`filter` cria um **novo array** sem alterar o original. Qualquer iteração em curso sobre o array antigo continua intacta. Quando ela termina, a variável `clientes` já aponta para o novo array sem o cliente removido. É uma operação imutável — mais segura em contexto assíncrono.

Adicionalmente, a arrow function da closure captura a referência de `ws` pelo escopo léxico — quando o evento `close` dispara, a variável `ws` ainda aponta para o socket correto, independente de quantas outras conexões entraram depois.

---

## Pergunta 8
**O que este trecho faz e por que o `e.preventDefault()` é necessário em ambos os botões?**

```javascript
const item = document.createElement("a")
item.href = "/uploads/" + arquivo.caminho

botaoExcluir.addEventListener("click", async (e) => {
    e.preventDefault()
    await fetch("/arquivo/" + arquivo.id, { method: "DELETE" })
})

botaoBaixar.addEventListener("click", (e) => {
    e.preventDefault()
    const link = document.createElement("a")
    link.href = "/uploads/" + arquivo.caminho
    link.download = arquivo.nome
    link.click()
})
```

### Resposta

**Contexto:** cada linha da tabela é criada como um elemento `<a>` (link) com `href` apontando para o arquivo. Isso permite que o usuário clique na linha inteira para baixar o arquivo.

**Por que `e.preventDefault()` no botão de excluir:**

Sem `preventDefault()`, clicar no botão de excluir dispararia **dois eventos**: o `click` no botão (que chama o DELETE) e, em seguida, o evento de clique se propagaria para o elemento `<a>` pai (bubble), navegando o browser para `href` — ou seja, tentando baixar o arquivo que acabou de ser deletado. `preventDefault()` cancela o comportamento padrão do `<a>` antes que isso aconteça.

**Por que `e.preventDefault()` no botão de baixar:**

Pelo mesmo motivo de propagação — sem isso, o clique subiria para o `<a>` pai e o browser navegaria para o `href` diretamente. O problema é que isso abriria o arquivo **no navegador** em vez de baixar (um PDF abriria no leitor do browser, uma imagem abriria na aba). O código cria programaticamente um `<a>` com o atributo `download`, que força o browser a baixar o arquivo com o **nome original** (`arquivo.nome`) em vez do nome com timestamp que existe no disco. Sem o `download`, o arquivo seria baixado com o nome `1716300000000-relatorio.pdf` — confuso para o usuário.

---

## Pergunta 9
**O que este trecho faz? Qual problema de distribuição ele resolve e qual é a sua limitação?**

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

### Resposta

**O que faz:**

Serializa o objeto `data` para uma string JSON (`JSON.stringify`) e envia para cada WebSocket no array `clientes` que ainda estiver com estado `OPEN`. `readyState` é uma propriedade do WebSocket que pode ser `CONNECTING(0)`, `OPEN(1)`, `CLOSING(2)` ou `CLOSED(3)` — só `OPEN` significa que a conexão está pronta para envio.

**Problema de distribuição que resolve:**

Implementa o padrão **Observer** distribuído: o servidor é o "subject" e todos os clientes conectados são os "observers". Quando o estado do sistema muda (novo arquivo ou arquivo deletado), todos os observers são notificados simultaneamente sem precisar perguntar. Isso é o que garante a sincronização em tempo real entre múltiplos clientes em computadores diferentes.

**Limitação crítica:**

Essa implementação só funciona se **todos os clientes estiverem conectados ao mesmo processo Node.js**. Em um sistema com múltiplos servidores (escalabilidade horizontal), um upload no servidor A envia broadcast apenas para os clientes conectados ao servidor A — os clientes do servidor B nunca saberiam do novo arquivo.

A solução para isso seria um **message broker** como Redis Pub/Sub: o servidor A publica o evento no Redis, e todos os servidores (A, B, C...) assinantes do Redis recebem e fazem broadcast para seus próprios clientes locais. Essa é uma das principais limitações arquiteturais do projeto para produção.

---

## Pergunta 10
**O que este trecho do `script.js` faz? Qual é a diferença entre este carregamento e o que acontece via WebSocket?**

```javascript
async function carregarArquivos() {
    const res = await fetch("/arquivos")
    const arquivos = await res.json()

    arquivos.forEach(arquivo => {
        criarItem(arquivo)
    })
}

carregarArquivos()
```

### Resposta

**O que faz:**

Ao carregar a página, faz uma requisição `GET /arquivos` ao servidor. O servidor retorna um array JSON com todos os arquivos existentes (filtrados pelo `existsSync`). Para cada arquivo, chama `criarItem()` que cria e insere a linha na tabela do DOM.

**A diferença em relação ao WebSocket:**

Este é o carregamento **inicial/estático**: acontece uma vez, quando a página abre, e recupera o estado atual do sistema (os arquivos que já existiam antes do usuário conectar).

O WebSocket é o canal **incremental/em tempo real**: após o carregamento inicial, qualquer novo arquivo ou exclusão que aconteça enquanto o usuário está na página é recebido via `socket.onmessage` sem nova requisição HTTP.

**Por que os dois são necessários:**

Se houvesse apenas WebSocket, um usuário que abre a página não veria nenhum arquivo — o WebSocket só entrega eventos futuros, não o estado passado. Se houvesse apenas `fetch`, o usuário precisaria recarregar a página manualmente para ver novidades.

Juntos, formam o padrão **"state + events"**: `fetch` traz o estado inicial, WebSocket mantém o estado sincronizado em tempo real.

---

# BLOCO 3 — "Explique o fluxo lógico"

---

## Pergunta 11
**"Dois usuários abrem o DailyManager ao mesmo tempo em computadores diferentes via ngrok. Usuário A faz upload de um arquivo. Descreva o fluxo completo, incluindo todos os protocolos envolvidos e o que acontece na tela do Usuário B."**

### Resposta

**Fase 1 — Estabelecimento de conexões (ambos os usuários)**

```
Usuário A                  ngrok              server.js
  │── GET / (HTTPS) ──────→│── GET / (HTTP) ─→│ Express serve index.html
  │←── index.html ─────────│←────────────────│
  │── GET /script.js ──────→│                 │
  │── Upgrade: websocket ──→│── ws:// ────────→│ wss.on("connection")
  │                         │                 │ clientes.push(wsA)
  
Usuário B (mesmo processo simultâneo)
  │── GET / (HTTPS) ──────→│── GET / (HTTP) ─→│ Express serve index.html
  │── Upgrade: websocket ──→│── ws:// ────────→│ wss.on("connection")
  │                         │                 │ clientes.push(wsB)
  │── GET /arquivos ───────→│                 │ retorna arquivos existentes
  │←── [] (vazio) ──────────│                 │
```

**Fase 2 — Upload pelo Usuário A**

```
Usuário A                  ngrok              server.js            Disco/BD
  │                         │                 │                    │
  │── POST /upload ─────────→│── POST ────────→│                    │
  │  (multipart, foto.jpg)  │                 │ Multer intercepta  │
  │                         │                 │──writeFile────────→│ uploads/1716..-foto.jpg
  │                         │                 │ db.prepare INSERT  │
  │                         │                 │──────────────────→ │ BD: id=1, nome=foto.jpg
  │                         │                 │                    │
  │                         │                 │ enviarParaTodos({tipo:"novo-arquivo"})
  │                         │                 │
  │←── "ok" ───────────────←│←── "ok" ────── │ res.send("ok")
```

**Fase 3 — Broadcast e atualização do DOM**

```
server.js              wsA (Usuário A)         wsB (Usuário B)
  │                        │                       │
  │── wsA.send(JSON) ─────→│ socket.onmessage()    │
  │                        │ msg.tipo="novo-arquivo"│
  │                        │ criarItem(msg.dados)   │
  │                        │ DOM: linha aparece     │
  │                        │                       │
  │── wsB.send(JSON) ──────────────────────────────→│ socket.onmessage()
  │                        │                       │ msg.tipo="novo-arquivo"
  │                        │                       │ criarItem(msg.dados)
  │                        │                       │ DOM: linha aparece
```

**Na tela do Usuário B:** sem recarregar a página, sem clicar em nada, a linha com `foto.jpg` aparece na tabela nos próximos milissegundos após o upload do Usuário A ser processado.

**Protocolos envolvidos:** HTTPS (TLS 1.3 → AES-256-GCM) entre browsers e ngrok; HTTP + WS entre ngrok e server.js (sem TLS, tráfego local); TCP em ambos os lados; SQLite sobre chamada de função local (sem rede).

---

## Pergunta 12
**"O servidor reinicia. Um usuário abre a página logo depois. Explique o que acontece com os dados — o que é perdido, o que é preservado, e por quê."**

### Resposta

**O que é preservado:**

**Metadados no banco SQLite:** o arquivo `database/database.db` é persistido em disco. O `CREATE TABLE IF NOT EXISTS` no início do `server.js` só cria a tabela se ela não existir — na reinicialização, a tabela e todos os registros existentes são mantidos intactos.

```javascript
const db = new Database("./database/database.db")  // abre ou cria
db.prepare(`CREATE TABLE IF NOT EXISTS arquivos(...)`)
  .run()  // só cria se não existir
```

**Arquivos físicos em disco:** o diretório `uploads/` e todos os arquivos nele são independentes do processo Node.js. Reiniciar o servidor não apaga arquivos do sistema operacional.

**O que é perdido:**

**Conexões WebSocket:** o array `clientes` é uma variável em memória (`let clientes = []`). Quando o processo encerra, a memória é liberada — todas as conexões WebSocket ativas são derrubadas. Os browsers dos usuários detectam isso (evento `close` no lado do cliente) e a conexão WebSocket fica morta.

**O que acontece quando o usuário abre a página após o restart:**

```
Usuário                    server.js (recém-iniciado)
  │── GET / ──────────────→│ Express serve index.html
  │── GET /script.js ─────→│
  │── Upgrade: websocket ──→│ nova conexão WS — clientes = [wsUsuario]
  │── GET /arquivos ───────→│ SELECT * FROM arquivos
  │                         │ filter(existsSync) — todos os arquivos válidos
  │←── [arquivo1, arquivo2]─│
  │ criarItem() para cada   │
  │ tabela populada         │
```

O usuário vê todos os arquivos que existiam antes — o estado é completamente restaurado pela combinação de SQLite (metadados) + disco (arquivos físicos). Para o usuário, é como se o servidor nunca tivesse reiniciado.

**Limitação:** usuários que estavam com a página aberta durante o restart precisam recarregar a página — o WebSocket morto não reconecta automaticamente (o projeto não implementa reconnect automático).

---

## Pergunta 13
**"Explique o fluxo completo da busca de arquivos. Por que ela funciona sem internet? O que seria necessário para transformá-la em uma busca server-side e quais seriam os trade-offs?"**

### Resposta

**Fluxo atual (client-side):**

```javascript
barraBuscar.addEventListener("input", function() {
    const busca = barraBuscar.value.toLowerCase()
    const arquivos = document.querySelectorAll(".linha-arquivo")

    arquivos.forEach(item => {
        const nome = item.innerText.toLowerCase()
        item.style.display = nome.includes(busca) ? "grid" : "none"
    })
})
```

1. Evento `input` dispara a cada tecla digitada.
2. Pega o texto da barra e converte para minúsculas.
3. Seleciona todos os elementos `.linha-arquivo` já presentes no DOM.
4. Para cada um, verifica se o texto do elemento inclui o termo buscado.
5. Esconde (`display: none`) ou mostra (`display: grid`) o elemento.

**Por que funciona sem internet:**

Toda a lógica opera exclusivamente no DOM do browser. Os elementos já foram carregados via `carregarArquivos()` ao abrir a página. A busca apenas altera a visibilidade de elementos que já estão em memória — nenhuma requisição de rede é feita.

**Para transformar em server-side:**

```javascript
// Client: enviar o termo para o servidor
barraBuscar.addEventListener("input", async function() {
    const busca = barraBuscar.value
    const res = await fetch(`/arquivos?busca=${encodeURIComponent(busca)}`)
    const arquivos = await res.json()
    listaArquivos.innerHTML = ""
    arquivos.forEach(criarItem)
})

// Server: filtrar no banco
app.get("/arquivos", (req, res) => {
    const busca = req.query.busca || ""
    const arquivos = db.prepare(
        "SELECT * FROM arquivos WHERE nome LIKE ?"
    ).all(`%${busca}%`)
    res.json(arquivos)
})
```

**Trade-offs:**

| | Client-side (atual) | Server-side |
|---|---|---|
| Latência | Zero (instantânea) | Depende da rede |
| Escala | Lento com milhares de itens no DOM | Banco indexado — rápido |
| Dados | Só busca o que já está carregado | Busca no banco completo |
| Tráfego | Zero | Requisição por tecla |
| Offline | Funciona | Não funciona |

Para o volume do projeto (dezenas de arquivos), client-side é a escolha correta. Com milhares de arquivos, o DOM ficaria pesado e server-side seria necessário.

---

## Pergunta 14
**"Explique o fluxo de como o WebSocket é estabelecido quando o usuário abre a página. Quais são as camadas de protocolo envolvidas e como o servidor atende HTTP e WebSocket na mesma porta?"**

### Resposta

**Por que HTTP e WebSocket compartilham a porta 3000:**

```javascript
const app    = express()
const server = http.createServer(app)        // servidor HTTP
const wss    = new WebSocket.Server({ server }) // WebSocket no MESMO server
server.listen(3000)
```

O WebSocket Server é criado com `{ server }` — ele "escuta" o evento `upgrade` do servidor HTTP. Quando chega uma requisição de upgrade, o `ws` assume o controle da conexão. Requisições normais continuam sendo tratadas pelo Express.

**Fluxo completo de handshake:**

```
Browser                                    server.js
  │                                             │
  │  1. TCP SYN ──────────────────────────────→ │
  │  ← TCP SYN-ACK ─────────────────────────── │
  │  2. TCP ACK ──────────────────────────────→ │
  │  (conexão TCP estabelecida)                 │
  │                                             │
  │  3. GET / HTTP/1.1                          │
  │     Host: localhost:3000                    │
  │     Upgrade: websocket          ──────────→ │ Express reconhece o Upgrade
  │     Connection: Upgrade                     │ ws intercepta
  │     Sec-WebSocket-Key: abc123=              │
  │     Sec-WebSocket-Version: 13               │
  │                                             │
  │  ← HTTP/1.1 101 Switching Protocols ─────── │
  │    Upgrade: websocket                       │
  │    Sec-WebSocket-Accept: xyz789=            │ (hash do Key + GUID RFC)
  │                                             │
  │  (protocolo muda para WebSocket)            │
  │  ════════════════════════════════════════   │
  │                                             │
  │  ← (servidor adiciona ws ao array clientes) │
  │                                             │
  │  4. script.js: socket.onopen → GET /arquivos│
  │     (fetch normal HTTP, porta 3000)  ──────→│ Express trata normalmente
  │  ← JSON com arquivos ──────────────────────│
```

**Camadas de protocolo empilhadas:**

```
Aplicação:   JSON (mensagens do sistema)
WebSocket:   frames WS (opcode, máscara, payload)
HTTP:        apenas no handshake inicial (101 Switching Protocols)
TLS:         (apenas via ngrok — o server local não tem TLS)
TCP:         entrega confiável, ordenada, sem perdas
IP:          roteamento (endereço de destino)
Ethernet/Wi-Fi: transmissão física
```

---

## Pergunta 15
**"Um atacante tenta um ataque de DDoS abrindo 10.000 conexões WebSocket simultâneas contra o servidor. Trace o fluxo do que acontece no sistema atual e explique por que o servidor eventualmente para de responder. Depois explique como o rate limiting resolveria isso."**

### Resposta

**O que acontece no sistema atual:**

```javascript
wss.on("connection", (ws) => {
    clientes.push(ws)   // sem verificação de limite
    // ...
})
```

Sem nenhuma proteção, cada conexão é aceita imediatamente.

**Fluxo do ataque:**

```
Atacante                               server.js (Node.js)
  │── TCP SYN (conexão 1) ────────────→│ aceita
  │── TCP SYN (conexão 2) ────────────→│ aceita
  │   ... (10.000 conexões) ...        │
  │── TCP SYN (conexão 10.000) ───────→│ aceita
  │                                    │
  │                                    │ clientes = [ws1, ws2, ..., ws10000]
  │                                    │ cada ws consome ~50KB de memória
  │                                    │ 10.000 × 50KB = 500MB de RAM
  │                                    │ limite de file descriptors do SO atingido
  │                                    │
  │── nova conexão (usuário legítimo) →│ RECUSADA: "EMFILE: too many open files"
```

**Por que para de responder:**

O sistema operacional limita o número de file descriptors por processo (padrão Linux: 1024). Cada socket TCP aberto consome um file descriptor. Com 10.000 conexões, o limite é atingido e novas conexões são recusadas — incluindo as de usuários legítimos. O servidor ainda está "rodando" mas inacessível.

Adicionalmente, a função `enviarParaTodos` itera por 10.000 entradas a cada upload — o que antes levava microssegundos passa a levar dezenas de milissegundos, bloqueando o Event Loop.

**Como o rate limiting resolveria:**

```javascript
const contagemPorIP = new Map()

wss.on("connection", (ws, req) => {
    const ip = req.socket.remoteAddress
    const agora = Date.now()
    const janela = 60000  // 1 minuto
    const limite = 5      // máximo 5 conexões WS por IP por minuto

    if (!contagemPorIP.has(ip)) contagemPorIP.set(ip, [])
    
    // remove timestamps fora da janela
    const historico = contagemPorIP.get(ip).filter(t => agora - t < janela)
    
    if (historico.length >= limite) {
        ws.close(1008, "Rate limit exceeded")  // recusa
        return
    }

    historico.push(agora)
    contagemPorIP.set(ip, historico)
    clientes.push(ws)
    // ...
})
```

**Fluxo com rate limiting:**

```
Atacante (mesmo IP)                    server.js
  │── conexão 1 ─────────────────────→ │ aceita (histórico: [t1])
  │── conexão 2 ─────────────────────→ │ aceita (histórico: [t1,t2])
  │── conexão 3 ─────────────────────→ │ aceita
  │── conexão 4 ─────────────────────→ │ aceita
  │── conexão 5 ─────────────────────→ │ aceita
  │── conexão 6 ─────────────────────→ │ RECUSADA: 1008 Rate limit exceeded
  │── conexão 7 ─────────────────────→ │ RECUSADA
  │   ... (todas recusadas)            │
  │                                    │
Usuário legítimo (IP diferente)        │
  │── conexão 1 ─────────────────────→ │ aceita normalmente
```

O servidor nunca acumula mais de `N × limite_por_IP` conexões no array. O ataque é mitigado na origem — antes de consumir recursos.

**Limitação:** um atacante sofisticado usa IPs diferentes (botnet distribuída — daí o "Distributed" em DDoS). Para isso, a proteção precisa estar na camada de rede (Cloudflare, firewall) antes de chegar ao processo Node.js.
