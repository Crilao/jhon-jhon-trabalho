# Holofix — Relatório do que foi feito

Este relatório explica **o que faltava**, **o que foi escrito** e **por que funciona assim**.

Ele foi escrito pensando em quem está começando a programar agora. Então, antes de mostrar
código, vou explicar as palavras difíceis. Se você já entende alguma parte, pode pular.

---

## Parte 1 — Entendendo o projeto

### 1.1 O que o Holofix faz

Imagine um técnico consertando uma máquina. Ele usa um óculos inteligente. Na lente do óculos
aparece o passo a passo do conserto. Se ele travar em algum passo, ele pede ajuda. Um especialista,
de outro lugar, vê o pedido num painel de computador e responde. A resposta aparece na lente.

Nesta hackathon o óculos não existe de verdade. Ele é **simulado**: uma página de navegador do
mesmo tamanho da lente, com o mesmo visual.

Então temos três personagens:

```
   [ ÓCULOS ]              [ SERVIDOR ]              [ PAINEL ]
  o técnico vê          app.py (o cérebro)       o especialista responde
   o passo a passo       e dados.py (a memória)
```

### 1.2 Por que existe um "servidor" no meio?

Uma pergunta justa: por que o óculos e o painel não conversam direto?

Porque eles são **computadores diferentes**, em lugares diferentes. O técnico está no galpão,
o especialista está no escritório. Os dois precisam de um lugar comum para guardar as informações.

Esse lugar comum é o **servidor**. Ele fica ligado o tempo todo, esperando pedidos.

Uma comparação que ajuda: pense num **restaurante**.

- Você (o óculos) não entra na cozinha. Você faz o pedido para o **garçom**.
- O garçom leva o pedido para a **cozinha**.
- A cozinha prepara e devolve o prato.
- O garçom traz o prato para você.

Nesse projeto:

| No restaurante | No Holofix |
|---|---|
| Você | O óculos (ou o painel) |
| O garçom | A **API** (as rotas do `app.py`) |
| A cozinha | O `dados.py` (onde as informações ficam guardadas) |
| O prato | O **JSON** (a resposta que volta) |

### 1.3 Glossário — as palavras que vão aparecer

Leia esta tabela com calma. Ela é a base de tudo.

| Palavra | O que significa, em português simples |
|---|---|
| **Servidor** | Um programa que fica ligado esperando pedidos e respondendo. |
| **API** | O conjunto de "portas de entrada" do servidor. Cada porta faz uma coisa. |
| **Rota** (ou *endpoint*) | Uma dessas portas. Ex.: `/api/maquinas` é a porta que devolve as máquinas. |
| **Requisição** (*request*) | O pedido que alguém faz ao servidor. |
| **Resposta** (*response*) | O que o servidor devolve. |
| **JSON** | Um formato de texto para organizar informação. É como uma ficha preenchida. |
| **GET** | Tipo de pedido que significa **"me dá"**. Só lê, não muda nada. |
| **POST** | Tipo de pedido que significa **"toma, guarda isso"**. Cria ou altera algo. |
| **HTTP** | O "idioma" que o navegador e o servidor usam para conversar. |
| **Código de status** | Um número que diz se deu certo ou errado. Ex.: 200 = deu certo. |
| **Dict** (dicionário) | Uma caixa com etiquetas. Cada etiqueta tem um nome e um valor. |
| **Lista** | Uma fila de coisas, em ordem. |
| **Função** | Um bloco de código com nome, que faz uma tarefa. |
| **`None`** | O jeito do Python de dizer "nada", "vazio", "não tem". |
| **`if`** | "Se acontecer isso, faça aquilo." |
| **`for`** | "Para cada item da lista, faça isso." |
| **`return`** | "Devolve este resultado e termina a função aqui." |

### 1.4 O que é um dict, na prática

Isso é importante porque quase tudo neste projeto é um dict. Olhe:

```python
maquina = {
    "codigo": "CMP-01",
    "nome": "Compressor Atlas",
    "local": "Galpão A",
    "procedimento_id": 1,
}
```

Pense numa **caixa de sapatos com etiquetas coladas**. A etiqueta `"codigo"` tem o valor
`"CMP-01"`. A etiqueta `"nome"` tem o valor `"Compressor Atlas"`.

Para pegar um valor, você usa o nome da etiqueta entre colchetes:

```python
maquina["nome"]        # devolve "Compressor Atlas"
```

### 1.5 O que é JSON, na prática

JSON é quase igual a um dict do Python, mas é **texto**. É o formato que viaja pela internet,
porque texto todo computador entende.

```json
{
  "codigo": "CMP-01",
  "nome": "Compressor Atlas"
}
```

Repare nas **aspas duplas** nos nomes das etiquetas. É uma das poucas diferenças do dict do Python.

O Flask tem uma função chamada `jsonify()` que faz essa conversão de graça:

```python
return jsonify(maquina)     # transforma o dict em JSON e devolve
```

### 1.6 Os códigos de status (os números 200, 404...)

Toda resposta de um servidor vem com um número. Esse número diz se deu certo. É como o
"código de rastreio" de uma entrega.

| Número | Nome | Tradução livre |
|---|---|---|
| **200** | OK | "Deu tudo certo, aqui está o que você pediu." |
| **201** | Created | "Deu certo, e eu **criei** uma coisa nova." |
| **400** | Bad Request | "Você pediu de um jeito errado. O problema foi no seu pedido." |
| **404** | Not Found | "Seu pedido foi feito direito, mas isso que você quer **não existe** aqui." |
| **501** | Not Implemented | "Eu ainda não sei fazer isso." (era o que o esqueleto devolvia) |

A diferença entre **400** e **404** confunde muito no começo. Vamos fixar com um exemplo real
deste projeto:

- O técnico manda `{"maquina_codigo": "CMP-01"}` mas **esquece de dizer o nome dele**.
  O pedido está mal formado → **400** ("você pediu errado").

- O técnico manda `{"maquina_codigo": "XYZ-99", "tecnico": "Ana"}`. O pedido está bem formado,
  mas a máquina `XYZ-99` não existe → **404** ("não achei isso").

Guarde assim:

> **400 = o problema está no que você mandou.**
> **404 = o que você mandou está certo, mas não existe.**

---

## Parte 2 — O que faltava fazer

### 2.1 O ponto de partida

O arquivo `app.py` veio como um **esqueleto**. Ele já tinha:

- As páginas prontas (`/oculos` e `/painel`)
- Funções auxiliares prontas (`erro`, `montar_ordem`, `total_de_passos`)
- Todas as rotas escritas, mas **vazias por dentro**

Cada rota vazia terminava assim:

```python
return nao_implementado(1)
```

Isso devolvia o código **501** com a mensagem *"Missão 1 ainda não implementada"*.

Ou seja: as páginas abriam, os botões apareciam, mas **nada funcionava**. Você clicava e
recebia um erro.

### 2.2 O que eu precisei escrever

Eram **12 tarefas** (marcadas como `TODO` no código), divididas em 5 missões:

| Missão | O que faz | Rotas |
|---|---|---|
| **1. Máquinas** | Listar e buscar máquinas | `GET /api/maquinas`<br>`GET /api/maquinas/<codigo>` |
| **2. Ordem de serviço** | Abrir e consultar uma manutenção | `POST /api/ordens`<br>`GET /api/ordens/<id>` |
| **3. Navegar pelos passos** | Avançar, voltar e concluir | `POST .../avancar`<br>`POST .../voltar`<br>`POST .../concluir` |
| **4. Pedir ajuda** (bônus) | O técnico chama o especialista | `POST /api/ordens/<id>/ajuda` |
| **5. Painel** (bônus) | O especialista responde | `GET /api/ajudas`<br>`POST /api/ajudas/<id>/responder`<br>`GET /api/ordens` |

A meta mínima da hackathon eram as missões **1, 2 e 3**. As missões 4 e 5 valiam pontos extras.

### 2.3 Resultado

Todas as 5 missões foram feitas. O resultado final:

```
[COMPLETA]  Missão 1 - Máquinas: 6/6
[COMPLETA]  Missão 2 - Abrir e consultar ordem de serviço: 7/7
[COMPLETA]  Missão 3 - Navegar pelos passos: 7/7
[COMPLETA]  Missão 4 - Pedir ajuda (bônus): 5/5
[COMPLETA]  Missão 5 - Painel do especialista (bônus): 7/7

Total: 32/32 verificações
Meta mínima (missões 1 a 3): 20/20
```

---

## Parte 3 — Missão por missão

A partir daqui, vou mostrar cada pedaço de código e explicar **linha por linha**.

---

### MISSÃO 1 — Máquinas

#### 1.1 — Listar todas as máquinas

**O que a rota precisa fazer:** devolver todas as máquinas numa lista.

No `dados.py`, as máquinas estão guardadas num dict:

```python
maquinas = {
    "CMP-01": {"codigo": "CMP-01", "nome": "Compressor Atlas", ...},
    "EST-02": {"codigo": "EST-02", "nome": "Esteira de embalagem", ...},
    "IMP-03": {"codigo": "IMP-03", "nome": "Impressora 3D do laboratório", ...},
}
```

Repare: a **chave** do dict é o código da máquina, e o **valor** é outro dict com os dados dela.

**O código que eu escrevi:**

```python
@app.route("/api/maquinas", methods=["GET"])
def listar_maquinas():
    return jsonify(list(dados.maquinas.values()))
```

Parece curto, mas tem duas coisas acontecendo. Vamos desmontar de dentro para fora:

**Passo 1 — `dados.maquinas.values()`**

O `.values()` pega só os **valores** do dict, jogando fora as chaves. Ou seja: pega só os
dados das máquinas, sem os códigos `"CMP-01"`, `"EST-02"`...

**Passo 2 — `list(...)`**

O `.values()` devolve uma "coisa" que não é bem uma lista. O `list(...)` transforma numa
lista de verdade, que o JSON entende.

**Passo 3 — `jsonify(...)`**

Transforma a lista em JSON e devolve para quem pediu.

> **Por que não devolver o dict inteiro?** Porque o óculos precisa **percorrer** a lista para
> desenhar um item embaixo do outro. Percorrer uma lista é natural. Percorrer um dict com
> chaves é mais complicado. Então convertemos para lista.

---

#### 1.2 — Buscar uma máquina pelo código

**O que a rota precisa fazer:** receber um código e devolver aquela máquina específica.

**O código que eu escrevi:**

```python
@app.route("/api/maquinas/<codigo>", methods=["GET"])
def buscar_maquina(codigo):
    codigo = codigo.strip().upper()
    if codigo not in dados.maquinas:
        return erro("Máquina não encontrada", 404)

    maquina = dados.maquinas[codigo]
    procedimento = dados.procedimentos[maquina["procedimento_id"]]

    resposta = dict(maquina)
    resposta["procedimento"] = procedimento["titulo"]
    resposta["total_passos"] = len(procedimento["passos"])
    return jsonify(resposta)
```

Vamos por partes.

**Parte A — Limpar o que o usuário digitou**

```python
codigo = codigo.strip().upper()
```

- `.strip()` tira espaços sobrando do começo e do fim.
  `"  cmp-01  "` vira `"cmp-01"`
- `.upper()` deixa tudo em maiúsculas.
  `"cmp-01"` vira `"CMP-01"`

**Por que isso importa?** Porque no `dados.py` a chave é `"CMP-01"` (maiúscula). Se o usuário
digitar `cmp-01` e a gente não arrumar, o Python não vai achar — para o Python, `"cmp-01"` e
`"CMP-01"` são textos **diferentes**.

> **Conceito:** isso se chama **normalizar a entrada**. É sempre melhor arrumar o dado assim que
> ele chega, na porta de entrada, e não ficar arrumando no meio do caminho. Assim o resto do
> código pode confiar que o dado já está limpo.

**Parte B — Verificar se existe antes de usar**

```python
if codigo not in dados.maquinas:
    return erro("Máquina não encontrada", 404)
```

O `not in` pergunta: "esse código **não** está entre as chaves do dict?"

**Por que essa checagem é obrigatória?** Porque se você tentar pegar uma chave que não existe,
o Python dá um erro chamado `KeyError` e o servidor devolve **500** (erro interno, feio, sem
explicação). Com a checagem, devolvemos um **404** limpo e explicativo.

> **Conceito:** um bom programa não deixa o erro acontecer — ele **prevê** o erro e responde
> direito.

**Parte C — Pegar a máquina e o procedimento dela**

```python
maquina = dados.maquinas[codigo]
procedimento = dados.procedimentos[maquina["procedimento_id"]]
```

Aqui tem uma **ligação entre duas tabelas**. Olhe:

- A máquina `CMP-01` tem `"procedimento_id": 1`
- Isso quer dizer: "o procedimento dela é o de número 1"
- Então vamos em `dados.procedimentos[1]` e achamos *"Troca do filtro de ar do compressor"*

É como uma **corrente**: a máquina aponta para o procedimento.

> **Conceito:** isso se chama **relacionamento**. Em vez de copiar o texto do procedimento
> dentro de cada máquina, a máquina só guarda o **número** do procedimento. Se o texto mudar,
> muda num lugar só. É assim que bancos de dados de verdade funcionam.

**Parte D — Montar a resposta (a parte mais importante)**

```python
resposta = dict(maquina)
resposta["procedimento"] = procedimento["titulo"]
resposta["total_passos"] = len(procedimento["passos"])
```

O `dict(maquina)` cria uma **cópia** da máquina.

**Por que copiar?** Essa é a parte que mais confunde no começo. Vamos com calma.

Imagine que o `dados.maquinas` é o **arquivo original** da escola. Se eu quiser entregar uma
cópia para o aluno, eu tiro um **xerox**.

```python
resposta = dict(maquina)      # tira um xerox
resposta["procedimento"] = ...  # rabisca no xerox
```

Agora, se eu **não** tivesse tirado o xerox:

```python
maquina["procedimento"] = ...   # ERRADO! escreve no arquivo original
```

Isso escreveria os campos novos **dentro do `dados.maquinas`**, estragando o "banco de dados"
para sempre. Toda vez que alguém buscasse essa máquina, ela viria com os campos extras já
grudados, e o dict iria crescendo sem parar.

**A regra:**

| Se você escreve... | O que acontece |
|---|---|
| `resposta = dict(maquina)` | Cópia. Mexer na cópia **não** afeta o original. |
| `resposta = maquina` | **Não é cópia!** São dois nomes para a mesma coisa. Mexer num, mexe no outro. |

Guarde esta frase:

> **`dict(x)` tira xerox. `= x` só dá um apelido.**

**Parte E — Devolver**

```python
return jsonify(resposta)
```

Transforma em JSON e devolve. Sem número de status, o Flask assume **200** (deu certo).

---

### MISSÃO 2 — Abrir e consultar ordem de serviço

#### O que é uma "ordem de serviço"?

É uma **manutenção acontecendo**. Ela guarda: quem é o técnico, qual a máquina, em que passo
ele está, quando começou e quando terminou.

No `dados.py`, as ordens começam vazias:

```python
ordens = {}        # id -> ordem
```

O comentário `id -> ordem` quer dizer: "um dict onde a chave é o número da ordem e o valor são
os dados dela".

---

#### 2.1 a 2.4 — Criar uma ordem

**O que a rota precisa fazer:** receber o código da máquina e o nome do técnico, e criar uma
ordem nova.

**O código que eu escrevi:**

```python
@app.route("/api/ordens", methods=["POST"])
def criar_ordem():
    corpo = request.get_json(silent=True) or {}
    codigo = (corpo.get("maquina_codigo") or "").strip().upper()
    tecnico = (corpo.get("tecnico") or "").strip()

    if not codigo or not tecnico:
        return erro("Informe maquina_codigo e tecnico", 400)
    if codigo not in dados.maquinas:
        return erro("Máquina não encontrada", 404)

    ordem = {
        "id": dados.proximo_id("ordem"),
        "maquina_codigo": codigo,
        "tecnico": tecnico,
        "status": "em_andamento",
        "passo_atual": 0,
        "inicio": dados.agora(),
        "fim": None,
    }
    dados.ordens[ordem["id"]] = ordem

    return jsonify(montar_ordem(ordem)), 201
```

Vamos desmontar. Essa rota tem mais partes, mas cada uma é simples.

**Parte A — Ler o que chegou**

```python
corpo = request.get_json(silent=True) or {}
```

Essa linha tem duas proteções. Vamos ver as duas.

**Primeira proteção: `silent=True`**

O `request.get_json()` lê o JSON que o cliente mandou. Mas se o cliente mandar algo que
**não é JSON** (ou não mandar nada), ele explode com um erro feio.

Com `silent=True`, em vez de explodir, ele devolve `None` (o "nada" do Python).

**Segunda proteção: `or {}`**

Agora `corpo` pode ser `None`. Se depois a gente tentar `corpo.get(...)`, dá erro.

O `or {}` resolve: em Python, `None or {}` devolve `{}` (um dict vazio).

Então a linha inteira significa:

> "Tenta ler o JSON. Se não conseguir, usa um dict vazio."

Assim, **nunca** dá erro aqui. Se o cliente mandar lixo, a gente simplesmente trata como
"não mandou nada" e responde 400 mais abaixo.

**Parte B — Pegar os campos (com a proteção do `or ""`)**

```python
codigo = (corpo.get("maquina_codigo") or "").strip().upper()
tecnico = (corpo.get("tecnico") or "").strip()
```

O `.get("tecnico")` tem um problema: se a chave `"tecnico"` **não existir** no dict, ele devolve
`None`. E aí `None.strip()` daria erro, porque `None` não é texto.

O `or ""` conserta: `None or ""` devolve `""`.

Então:

| O cliente mandou | `corpo.get("tecnico")` | `or ""` dá | `.strip()` dá |
|---|---|---|---|
| `{"tecnico": "Ana"}` | `"Ana"` | `"Ana"` | `"Ana"` |
| `{"tecnico": "  Ana  "}` | `"  Ana  "` | `"  Ana  "` | `"Ana"` |
| `{}` (sem técnico) | `None` | `""` | `""` |

No último caso, `tecnico` vira `""` (texto vazio) e a gente consegue detectar o problema sem
o programa quebrar.

> **Conceito importante:** em Python, um texto vazio `""` é considerado **falso**.
> Então `if not tecnico:` é verdadeiro quando `tecnico` é `""`. É por isso que essa checagem
> funciona para pegar campo em branco.

**Parte C — Validar (na ordem certa!)**

```python
if not codigo or not tecnico:
    return erro("Informe maquina_codigo e tecnico", 400)
if codigo not in dados.maquinas:
    return erro("Máquina não encontrada", 404)
```

**A ordem aqui importa.** Repare:

1. Primeiro checamos se os campos estão **preenchidos** → se não, **400**
2. Depois checamos se a máquina **existe** → se não, **404**

Se a ordem fosse invertida, um pedido sem técnico e com máquina inexistente devolveria 404
quando o problema real era o pedido malformado.

**Por que `return` no meio da função?** Porque o `return` **encerra a função na hora**. Se a
validação falha, o código para ali e não chega a criar a ordem. Isso evita ficar usando
`if/else` aninhado, que deixa o código difícil de ler.

> **Conceito:** isso se chama *"early return"* (retorno cedo). É um jeito muito comum e limpo
> de escrever validações.

**Parte D — Criar a ordem**

```python
ordem = {
    "id": dados.proximo_id("ordem"),
    "maquina_codigo": codigo,
    "tecnico": tecnico,
    "status": "em_andamento",
    "passo_atual": 0,
    "inicio": dados.agora(),
    "fim": None,
}
dados.ordens[ordem["id"]] = ordem
```

- `dados.proximo_id("ordem")` devolve 1, depois 2, depois 3... É o contador que dá um número
  novo para cada ordem. Assim duas ordens nunca têm o mesmo número.
- `"passo_atual": 0` → **o primeiro passo é o 0, não o 1!** Isso é muito importante, volto
  nisso já já.
- `"inicio": dados.agora()` → pega a data e hora atual, tipo `"2026-10-22 14:32:10"`.
- `"fim": None` → a ordem ainda não terminou, então não tem data de fim. `None` = "nada".
- A última linha guarda a ordem no "banco de dados".

**Sobre o passo começar em 0**

Isso confunde todo mundo no começo, mas é assim que programação funciona: **contagem começa
em zero**.

Se um procedimento tem 4 passos, eles são numerados `0, 1, 2, 3`. Não `1, 2, 3, 4`.

Por isso, em vários lugares do projeto você vai ver `passo_atual + 1`. Isso existe porque
**para o humano** mostramos "Passo 1 de 4", mas **por dentro** o número guardado é 0.

**Parte E — Devolver com 201**

```python
return jsonify(montar_ordem(ordem)), 201
```

Duas coisas:

1. `montar_ordem(ordem)` — explico essa função daqui a pouco, é o coração do projeto
2. `, 201` — devolve o código **201 Created** ("criei uma coisa nova")

Repare na vírgula: `return jsonify(x), 201` devolve **dois valores**: o JSON e o número 201.
O Flask entende isso.

---

#### 2.5 — Consultar uma ordem

**O código que eu escrevi:**

```python
@app.route("/api/ordens/<int:ordem_id>", methods=["GET"])
def buscar_ordem(ordem_id):
    ordem = dados.ordens.get(ordem_id)
    if ordem is None:
        return erro("Ordem não encontrada", 404)
    return jsonify(montar_ordem(ordem))
```

**Novidade aqui: `.get()` em vez de `[...]`**

Na Missão 1 eu usei `dados.maquinas[codigo]`, com colchetes. Aqui usei `dados.ordens.get(...)`.

Qual a diferença?

| Jeito | Se a chave não existir |
|---|---|
| `dados.ordens[999]` | 💥 Erro `KeyError`, servidor devolve 500 |
| `dados.ordens.get(999)` | Devolve `None` calmamente |

**Quando usar cada um?**

- Use `.get()` quando "não existir" é uma situação **normal** que você precisa tratar.
  (Aqui: o usuário pode pedir uma ordem que não existe → queremos dar 404.)
- Use `[...]` quando você **já validou** que existe, ou quando não existir seria um bug de
  verdade que você quer descobrir.

E o `<int:ordem_id>` na rota? Isso diz ao Flask: "esse pedaço da URL é um número inteiro".
Se alguém acessar `/api/ordens/abc`, o Flask nem chama a função — devolve 404 sozinho.

---

### MISSÃO 3 — Navegar pelos passos

Esta missão tem 3 rotas, e as três seguem **a mesma receita**. O próprio código já dizia isso
num comentário:

```
1. Busca a ordem            -> se não achou: 404
2. Já está concluída?       -> se sim: 400
3. Regra específica         -> se não pode: 400
4. Altera e devolve
```

#### 3.1 — Avançar

```python
@app.route("/api/ordens/<int:ordem_id>/avancar", methods=["POST"])
def avancar_passo(ordem_id):
    ordem = dados.ordens.get(ordem_id)
    if ordem is None:
        return erro("Ordem não encontrada", 404)
    if ordem["status"] == "concluida":
        return erro("Ordem já concluída", 400)
    if ordem["passo_atual"] >= total_de_passos(ordem) - 1:
        return erro("Já está no último passo", 400)

    ordem["passo_atual"] += 1
    return jsonify(montar_ordem(ordem))
```

**A parte que precisa de atenção:**

```python
if ordem["passo_atual"] >= total_de_passos(ordem) - 1:
```

Vamos traduzir: *"se o passo atual for maior ou igual ao último passo possível"*.

O último passo é `total - 1`. Por quê? Lembra que a contagem começa em 0?

- 4 passos → passos `0, 1, 2, 3` → o último é `4 - 1 = 3`

**O que aconteceria se eu esquecesse o `- 1`?**

O técnico estaria no passo 3 (o último). A checagem `3 >= 4` seria **falsa**, então o código
deixaria avançar para o passo 4. Mas o passo 4 **não existe**! Aí, lá dentro do `montar_ordem`,
que faz `passos[ordem["passo_atual"]]`, o Python daria erro `IndexError` (índice fora da lista)
e o servidor devolveria 500.

> **Conceito:** esse tipo de erro tem nome: **erro de "um a mais"** (ou *off-by-one*). É um dos
> erros mais comuns da programação inteira. Sempre que você mexer com listas, pergunte:
> *"meu último índice é `tamanho` ou `tamanho - 1`?"*

**E o `+= 1`?**

```python
ordem["passo_atual"] += 1
```

Isso é um atalho para `ordem["passo_atual"] = ordem["passo_atual"] + 1`. Soma 1 no passo.

**Importante:** aqui a gente **altera** a ordem de verdade (não é cópia!). E isso é
intencional — queremos que a mudança fique salva.

> Isso acontece porque `dados.ordens.get(ordem_id)` devolve uma **referência** para o dict
> guardado, não um xerox. Voltando à comparação: aqui não é xerox, é **a mesma folha de papel**.
> Se eu escrevo nela, está escrito para sempre.

#### 3.2 — Voltar

```python
    if ordem["passo_atual"] <= 0:
        return erro("Já está no primeiro passo", 400)

    ordem["passo_atual"] -= 1
```

Mesma ideia, ao contrário. Não pode voltar se já está no passo 0. O `-= 1` subtrai 1.

#### 3.3 — Concluir

```python
    if ordem["passo_atual"] < total_de_passos(ordem) - 1:
        return erro("Ainda faltam passos", 400)

    ordem["status"] = "concluida"
    ordem["fim"] = dados.agora()
    return jsonify(montar_ordem(ordem))
```

Só pode concluir se estiver **no último passo**. Ao concluir:

- `status` vira `"concluida"`
- `fim` recebe a data e hora de agora

**Repare na ordem das checagens nesta rota.** A checagem de "já concluída" vem **antes** da
checagem do último passo:

```python
    if ordem["status"] == "concluida":      # <- vem primeiro
        return erro("Ordem já concluída", 400)
    if ordem["passo_atual"] < total_de_passos(ordem) - 1:   # <- vem depois
        return erro("Ainda faltam passos", 400)
```

**Por que essa ordem?** Imagine que ela estivesse invertida. O técnico conclui a ordem. Agora
ele chama `concluir` de novo, sem querer:

- A ordem continua no último passo (concluir não muda o passo)
- Então a checagem do passo **passaria**
- E o código gravaria uma **nova** data de fim, apagando a data verdadeira

Com a ordem correta, a segunda chamada bate na primeira checagem e devolve 400.

> **Conceito:** uma ordem concluída é um **registro histórico**. Ela não deve mais mudar. Isso
> se chama **imutabilidade** — depois que algo terminou, vira só leitura.

---

### MISSÃO 4 — Pedir ajuda ao especialista (bônus)

**O que a rota precisa fazer:** o técnico manda uma mensagem, e isso vira um "pedido de ajuda"
que o especialista vai ver no painel.

**O código que eu escrevi:**

```python
@app.route("/api/ordens/<int:ordem_id>/ajuda", methods=["POST"])
def pedir_ajuda(ordem_id):
    ordem = dados.ordens.get(ordem_id)
    if ordem is None:
        return erro("Ordem não encontrada", 404)
    if ordem["status"] == "concluida":
        return erro("Ordem já concluída", 400)

    corpo = request.get_json(silent=True) or {}
    mensagem = (corpo.get("mensagem") or "").strip()
    if not mensagem:
        return erro("Escreva a mensagem", 400)

    pedido = {
        "id": dados.proximo_id("ajuda"),
        "ordem_id": ordem_id,
        "passo": ordem["passo_atual"],
        "mensagem": mensagem,
        "resposta": None,
        "status": "aberto",
        "criado_em": dados.agora(),
        "respondido_em": None,
    }
    dados.pedidos_ajuda[pedido["id"]] = pedido

    ordem["status"] = "aguardando_ajuda"

    return jsonify(pedido), 201
```

**Os pontos que merecem explicação:**

**1. `"passo": ordem["passo_atual"]` — por que guardar isso?**

O especialista precisa saber **onde** o técnico travou. Se o técnico está no passo 3 e diz
"não acho a peça", o especialista precisa saber que é o passo 3.

Aqui a gente **copia o número** do passo no momento do pedido. Poderia parecer melhor ler o
passo atual na hora de mostrar, mas aí teria um problema: se o técnico continuasse navegando,
o especialista veria um passo diferente do que o técnico estava quando pediu ajuda. Guardando
o número, o pedido fica **congelado** como estava.

**2. `ordem["status"] = "aguardando_ajuda"` — a máquina de estados**

O status da ordem vai mudando conforme a vida dela:

```
   em_andamento  ──(técnico pede ajuda)──>  aguardando_ajuda
        ^                                          │
        └────────(especialista responde)───────────┘
        │
        └──(técnico conclui)──>  concluida
```

Isso se chama **máquina de estados**: um conjunto limitado de situações, com regras de qual
situação pode ir para qual.

**3. Devolve o `pedido`, não o `montar_ordem`**

Repare que aqui o retorno é `jsonify(pedido)`, e não `jsonify(montar_ordem(ordem))`.

Isso porque **esta rota devolve um pedido de ajuda**, não uma ordem. Cada rota devolve o tipo
de coisa que faz sentido para ela. Misturar os dois seria confuso para o óculos.

---

### MISSÃO 5 — Painel do especialista (bônus)

#### 5.1 — Listar os pedidos de ajuda

**O código que eu escrevi:**

```python
@app.route("/api/ajudas", methods=["GET"])
def listar_ajudas():
    filtro = request.args.get("status")

    lista = []
    for pedido in dados.pedidos_ajuda.values():
        if filtro and pedido["status"] != filtro:
            continue

        item = dict(pedido)
        item["ordem"] = montar_ordem(dados.ordens[pedido["ordem_id"]])
        lista.append(item)

    return jsonify(lista)
```

**Novidade 1 — `request.args` em vez de `request.get_json`**

Até agora a gente lia o que vinha no **corpo** do pedido (`request.get_json()`). Aqui a gente
lê o que vem na **URL**.

Olhe a diferença:

| Onde o dado viaja | Como ler | Exemplo |
|---|---|---|
| No corpo (POST) | `request.get_json()` | `POST /api/ordens` com `{"tecnico": "Ana"}` |
| Na URL (GET) | `request.args.get()` | `GET /api/ajudas?status=aberto` |

O `?status=aberto` no fim da URL se chama **query string** ("string de consulta"). É o jeito
padrão de passar filtros num GET.

**Novidade 2 — o `for`**

```python
lista = []
for pedido in dados.pedidos_ajuda.values():
    ...
    lista.append(item)
```

Isso se lê assim: *"para cada pedido dentro dos pedidos de ajuda, faça o que está aqui dentro"*.

O padrão é sempre o mesmo:

1. Cria uma lista vazia **antes** do `for`
2. Dentro do `for`, adiciona itens com `.append(...)`
3. Depois do `for`, usa a lista

**Novidade 3 — o filtro opcional**

```python
filtro = request.args.get("status")
if filtro and pedido["status"] != filtro:
    continue
```

Vamos por partes:

- Se a URL **não** tem `?status=...`, o `filtro` é `None`
- `None` é falso em Python, então `if filtro and ...` já é falso → **não pula nada** (mostra tudo)
- Se a URL **tem** `?status=aberto`, o `filtro` é `"aberto"`
- Aí a segunda parte é checada: se o pedido não for daquele status, `continue` **pula** ele

O `continue` significa *"pula para o próximo item do for, sem executar o resto"*.

> **Conceito:** `if filtro and ...` é um jeito bem comum de escrever "filtro opcional" em
> Python. Como `None` já é falso, não precisa escrever `if filtro is not None and ...`.

**Novidade 4 — juntar o pedido com a ordem**

```python
item = dict(pedido)
item["ordem"] = montar_ordem(dados.ordens[pedido["ordem_id"]])
```

De novo: `dict(pedido)` é **xerox**. A gente copia o pedido e cola um campo novo dentro da cópia.

Esse campo novo é a **ordem inteira** ligada àquele pedido. É assim que o painel consegue
mostrar o nome do técnico e a máquina (`p.ordem.tecnico`, `p.ordem.maquina_codigo`).

Sem isso, o painel mostraria só a mensagem solta, sem dizer de quem era.

#### 5.2 a 5.5 — Responder um pedido

**O código que eu escrevi:**

```python
@app.route("/api/ajudas/<int:ajuda_id>/responder", methods=["POST"])
def responder_ajuda(ajuda_id):
    pedido = dados.pedidos_ajuda.get(ajuda_id)
    if pedido is None:
        return erro("Pedido não encontrado", 404)
    if pedido["status"] == "respondido":
        return erro("Pedido já respondido", 400)

    corpo = request.get_json(silent=True) or {}
    resposta = (corpo.get("resposta") or "").strip()
    if not resposta:
        return erro("Escreva a resposta", 400)

    pedido["resposta"] = resposta
    pedido["status"] = "respondido"
    pedido["respondido_em"] = dados.agora()

    ordem = dados.ordens.get(pedido["ordem_id"])
    if ordem is not None and ordem["status"] == "aguardando_ajuda":
        ordem["status"] = "em_andamento"

    return jsonify(pedido)
```

A parte interessante é o final:

```python
if ordem is not None and ordem["status"] == "aguardando_ajuda":
    ordem["status"] = "em_andamento"
```

**Traduzindo:** *"se a ordem existe E ela está esperando ajuda, devolve ela para em andamento"*.

**Por que checar se está `aguardando_ajuda` antes de mudar?**

Porque a ordem pode não estar mais esperando. Imagine que o especialista demorou, e o técnico
resolveu sozinho e continuou. Se a gente mudasse o status sem checar, estaríamos **atropelando**
o estado real da ordem.

**Regra geral:** só mude o estado de algo se ele estiver no estado que você espera. Isso evita
que duas partes do sistema briguem entre si.

#### 5.6 — Listar todas as ordens

```python
@app.route("/api/ordens", methods=["GET"])
def listar_ordens():
    return jsonify([montar_ordem(o) for o in dados.ordens.values()])
```

O pedaço `[montar_ordem(o) for o in dados.ordens.values()]` é uma **list comprehension**
(compreensão de lista). É um jeito curto de escrever um `for` que constrói uma lista.

Compare os dois — fazem exatamente a mesma coisa:

```python
# Jeito longo
lista = []
for o in dados.ordens.values():
    lista.append(montar_ordem(o))

# Jeito curto (list comprehension)
lista = [montar_ordem(o) for o in dados.ordens.values()]
```

O jeito curto se lê: *"para cada `o` em ordens, me dá `montar_ordem(o)`, tudo numa lista"*.

> **Dica:** no começo, se a versão curta te confundir, escreva a versão longa. As duas estão
> certas. A curta é só mais comum de ver por aí.

---

## Parte 4 — O coração do projeto: `montar_ordem`

Essa função **já vinha pronta** no esqueleto. Você não precisa mexer nela. Mas entender ela é
o que faz o projeto inteiro fazer sentido. Então vamos com calma.

### 4.1 O problema que ela resolve

Quando a gente guarda uma ordem, guardamos só **números e códigos**:

```python
ordem = {
    "id": 1,
    "maquina_codigo": "CMP-01",     # só o código, não o nome
    "tecnico": "Ana",
    "passo_atual": 2,               # só o número, não o texto
    "status": "em_andamento",
    "inicio": "2026-10-22 14:32:10",
    "fim": None,
}
```

Mas o óculos precisa mostrar **texto legível**:

- O nome da máquina: *"Compressor Atlas"* (não `CMP-01`)
- O título do procedimento: *"Troca do filtro de ar do compressor"*
- O texto do passo: *"Abra a tampa lateral soltando os 4 parafusos."* (não `2`)

**Quem faz essa tradução?** A função `montar_ordem`.

### 4.2 Como ela funciona

```python
def montar_ordem(ordem):
    maquina = dados.maquinas[ordem["maquina_codigo"]]
    procedimento = dados.procedimentos[maquina["procedimento_id"]]
    passos = procedimento["passos"]
    ...
```

Ela vai **seguindo a corrente de ligações**:

```
ordem["maquina_codigo"] = "CMP-01"
        ↓
dados.maquinas["CMP-01"] = { ..., "procedimento_id": 1 }
        ↓
dados.procedimentos[1] = { "titulo": "...", "passos": [ ... ] }
        ↓
passos[ordem["passo_atual"]] = "Abra a tampa lateral..."
```

Cada seta é uma consulta. No fim, ela devolve tudo junto num dict só, já mastigado.

> **Conceito:** em bancos de dados isso se chama **JOIN** — juntar dados de tabelas diferentes
> que estão ligadas por um número. Aqui estamos fazendo isso na mão, com dicts.

### 4.3 A parte mágica: como a resposta chega no óculos

Esta é a parte mais bonita do projeto. Olhe as últimas linhas do `montar_ordem`:

```python
    ultima_ajuda = None
    for pedido in dados.pedidos_ajuda.values():
        if pedido["ordem_id"] == ordem["id"]:
            ultima_ajuda = pedido

    return {
        ...
        "ajuda": ultima_ajuda,
    }
```

A função procura o pedido de ajuda daquela ordem e coloca ele no campo `"ajuda"`.

Agora a pergunta: **o que é colocado ali?** Um xerox do pedido, ou o pedido de verdade?

**É o pedido de verdade.** É a **mesma folha de papel**.

E é exatamente por isso que tudo funciona sem nenhuma complicação extra. Veja a sequência:

1. O óculos pergunta ao servidor a cada 3 segundos: *"como está a ordem 1?"*
2. O servidor roda `montar_ordem`, que devolve uma referência ao pedido de ajuda
3. O especialista responde, e o código faz `pedido["resposta"] = "Olhe atrás do painel"`
4. **Como é a mesma folha**, a resposta já está lá dentro
5. Na próxima pergunta (no máximo 3 segundos depois), o óculos recebe a resposta

**Não existe nenhum sistema de notificação, nenhum "avise o óculos".** A resposta aparece
simplesmente porque os dois estão olhando para o mesmo objeto.

> **Conceito:** isso se chama **compartilhamento por referência**. É poderoso, mas tem que
> tomar cuidado — quando duas partes do programa mexem na mesma coisa, é fácil se confundir.
> Foi exatamente por isso que, nas rotas, a gente usou `dict(...)` para tirar xerox antes de
> adicionar campos.

### 4.4 Como o óculos sabe qual pedido mostrar

```python
    for pedido in dados.pedidos_ajuda.values():
        if pedido["ordem_id"] == ordem["id"]:
            ultima_ajuda = pedido
```

Esse `for` percorre **todos** os pedidos e vai ficando com o último que combina. Como o Python
guarda dicts na ordem em que foram criados, o "último que combina" é o **mais recente**.

Ou seja: se o técnico pediu ajuda três vezes, o óculos mostra a mais recente.

---

## Parte 5 — Conceitos para levar para a vida

### 5.1 Cópia x Referência (o conceito mais importante daqui)

| Código | O que faz | Analogia |
|---|---|---|
| `dict(maquina)` | **Cópia** | Tirar xerox. Rabiscar a xerox não muda o original. |
| `maquina` | **Referência** | Dois nomes para a mesma folha. Um escreve, o outro vê. |

Neste projeto usamos os dois, de propósito:

- **Xerox** quando vamos **adicionar campos** para a resposta (Missões 1, 5.1) — para não
  estragar o banco de dados
- **Referência** quando vamos **alterar de verdade** (Missões 3, 4, 5.2) — porque queremos que
  a mudança fique salva

### 5.2 Mutação — alterar sem criar

```python
ordem["status"] = "concluida"
```

Isso **não** cria uma ordem nova. Altera a que já existe. Em Python, dicts e listas são
**mutáveis** (podem ser alterados). Textos e números não são — se você fizer
`nome = nome.upper()`, está criando um texto novo, não alterando o antigo.

### 5.3 Validação na porta de entrada

Sempre que um dado vem de fora (do usuário, da internet), ele pode vir errado. Então:

1. **Limpe** o dado (`.strip()`, `.upper()`)
2. **Confira** se está preenchido
3. **Confira** se faz sentido (a máquina existe?)
4. Só então **use**

E faça isso **logo na entrada**, não no meio do código.

### 5.4 Prever o erro em vez de sofrer com ele

Compare:

```python
# Ruim: deixa o erro acontecer
maquina = dados.maquinas[codigo]      # 💥 KeyError se não existir

# Bom: prevê e responde direito
if codigo not in dados.maquinas:
    return erro("Máquina não encontrada", 404)
maquina = dados.maquinas[codigo]
```

### 5.5 Retorno cedo (early return)

Em vez de aninhar vários `if/else`, vá "cortando" os casos inválidos e retornando logo:

```python
if ordem is None:
    return erro("Ordem não encontrada", 404)
if ordem["status"] == "concluida":
    return erro("Ordem já concluída", 400)
if ordem["passo_atual"] <= 0:
    return erro("Já está no primeiro passo", 400)

# Se chegou aqui, está tudo certo
ordem["passo_atual"] -= 1
return jsonify(montar_ordem(ordem))
```

Isso é bem mais fácil de ler do que:

```python
# Ruim: aninhado, difícil de acompanhar
if ordem is not None:
    if ordem["status"] != "concluida":
        if ordem["passo_atual"] > 0:
            ordem["passo_atual"] -= 1
            return jsonify(montar_ordem(ordem))
```

### 5.6 Contagem começa em zero

Sempre. O primeiro item de uma lista é `lista[0]`. Isso vale para quase toda linguagem de
programação. É a fonte número um de erros de iniciante — e não é culpa sua, é estranho mesmo.

---

## Parte 6 — Como rodar e conferir

### 6.1 Instalar e rodar

```bash
cd holofix-main

# cria um "ambiente isolado" para não sujar o Python do computador
python3 -m venv .venv

# instala o Flask dentro desse ambiente
.venv/bin/pip install -r requirements.txt

# liga o servidor
.venv/bin/python app.py
```

Abra no navegador: **`http://127.0.0.1:5000/`**

Você vai ver três links:

- **`/oculos`** — a simulação do óculos (use as setas do teclado e o Enter)
- **`/painel`** — o painel do especialista
- **`/api/maquinas`** — o JSON cru, para ver como é por dentro

### 6.2 Rodar os testes

Em **outro terminal** (deixe o servidor rodando):

```bash
.venv/bin/python verificar.py       # testa todas as missões
.venv/bin/python verificar.py 3     # testa só a missão 3
```

### 6.3 Um problema do macOS (não é bug do código!)

No Mac, o **AirPlay** também usa a porta 5000. Então `localhost:5000` pode acabar batendo no
AirPlay em vez do seu servidor, e aí volta uma resposta estranha (não é JSON), e a página diz
*"essa rota não existe"*.

**Solução:** use `127.0.0.1:5000` em vez de `localhost:5000`. Ou desligue o AirPlay em
Ajustes do Sistema.

Isso me deu uma boa confusão quando eu estava testando, então deixo o aviso aqui.

### 6.4 Testando na mão (com o servidor ligado)

Se quiser brincar com a API direto, sem usar a página:

```bash
B=http://127.0.0.1:5000

# 1) o técnico abre uma ordem
curl -s -X POST $B/api/ordens -H 'Content-Type: application/json' \
  -d '{"maquina_codigo":"CMP-01","tecnico":"Ana"}'

# 2) avança um passo
curl -s -X POST $B/api/ordens/1/avancar

# 3) o técnico pede ajuda
curl -s -X POST $B/api/ordens/1/ajuda -H 'Content-Type: application/json' \
  -d '{"mensagem":"Nao encontro a peca"}'

# 4) o especialista vê o pedido
curl -s "$B/api/ajudas?status=aberto"

# 5) o especialista responde
curl -s -X POST $B/api/ajudas/1/responder -H 'Content-Type: application/json' \
  -d '{"resposta":"Olhe atras do painel lateral"}'

# 6) o óculos agora vê a resposta
curl -s $B/api/ordens/1
```

No último comando, procure por `"ajuda"` no JSON. Vai ter a resposta lá dentro.

---

## Parte 7 — O que foi conferido

### 7.1 Os testes automáticos

```
[COMPLETA]  Missão 1 - Máquinas: 6/6
[COMPLETA]  Missão 2 - Abrir e consultar ordem de serviço: 7/7
[COMPLETA]  Missão 3 - Navegar pelos passos: 7/7
[COMPLETA]  Missão 4 - Pedir ajuda (bônus): 5/5
[COMPLETA]  Missão 5 - Painel do especialista (bônus): 7/7

Total: 32/32 verificações
Meta mínima (missões 1 a 3): 20/20
```

### 7.2 Testes manuais que eu fiz além dos automáticos

Testei com o servidor de verdade rodando, para garantir que funciona fora do teste automático:

- Digitar `cmp-01` em minúsculas funciona (é normalizado para `CMP-01`)
- Digitar `"  Ana  "` com espaços funciona (os espaços são removidos)
- Tentar **voltar** no primeiro passo → erro 400
- Tentar **avançar** depois do último passo → erro 400
- **Concluir** no último passo → vira `concluida` e ganha data de fim
- **Concluir** duas vezes → erro 400 *"Ordem já concluída"*
- O ciclo completo de ajuda:
  pedido → ordem vira `aguardando_ajuda` → resposta → ordem volta para `em_andamento`
  → **e o óculos lê a resposta**
- O filtro `?status=aberto` não mostra os já respondidos
- Responder o mesmo pedido duas vezes → erro 400 *"Pedido já respondido"*

---

## Parte 8 — Arquivos alterados

| Arquivo | O que mudou |
|---|---|
| `app.py` | Implementadas as 12 tarefas das 5 missões |
| `.gitignore` | **Novo** — para o Git ignorar a pasta `.venv/` e o `__pycache__/` |
| `RELATORIO.md` | **Novo** — este relatório |

**Não foram alterados:** `dados.py`, `verificar.py` e tudo dentro de `static/`.

O `README.md` pedia explicitamente para não alterar esses arquivos, então foram deixados em paz.

> **O que é `.gitignore`?** É um arquivo que diz ao Git quais arquivos ele deve **ignorar**.
> A pasta `.venv/` é o ambiente que a gente criou, e ela é só cópia de bibliotecas — não faz
> sentido mandar isso para o repositório. Se você não ignorar, ela vai junto e deixa o
> repositório enorme à toa.

---

## Parte 9 — Ideias para continuar

O esqueleto falava em "desafios extras" mas não dava nenhum `TODO`. Aqui vão sugestões, do
mais fácil para o mais difícil:

**1. Salvar os dados em arquivo (fácil)**
Hoje, se o servidor reiniciar, tudo some. Você poderia salvar as ordens num arquivo `.json`
e carregar de volta quando o servidor liga.

**2. Histórico por máquina (fácil)**
Uma rota nova que mostra todas as manutenções já feitas numa máquina:
`GET /api/maquinas/CMP-01/historico`

**3. Calcular quanto tempo durou (médio)**
Você já tem `inicio` e `fim` salvos. Dá para calcular a diferença e mostrar
*"essa manutenção levou 42 minutos"*.

**4. Cuidado com dois técnicos ao mesmo tempo (difícil)**
O servidor do Flask atende várias pessoas ao mesmo tempo. Se dois técnicos mexerem no mesmo
dict no mesmo instante, pode dar confusão. A solução se chama `threading.Lock` — mas isso é
assunto para depois que você estiver confortável com o básico.

**Sugestão:** comece pela 1 ou pela 2. São as que mais ensinam e as que dão menos dor de cabeça.

---

## Resumo final

O que estava faltando era **a lógica do servidor**. As páginas já existiam, mas não tinham com
quem conversar.

O trabalho foi escrever **12 pedaços de código** que respondem a 12 perguntas:

1. Quais são as máquinas?
2. Como é a máquina tal?
3. Cria uma ordem de serviço
4. Como está a ordem tal?
5. Avança um passo
6. Volta um passo
7. Conclui a ordem
8. Técnico pede ajuda
9. Quais pedidos de ajuda existem?
10. Especialista responde
11. Quais ordens existem?
12. (e as validações de tudo isso)

E no meio disso, os conceitos que mais apareceram foram: **validar antes de usar**, **copiar
antes de alterar**, **prever o erro em vez de deixar acontecer**, e **contagem começando em
zero**.

Se você entendeu esses quatro, entendeu o projeto.
