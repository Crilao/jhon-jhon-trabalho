# Holofix — Report on the Completed Work

This report explains **what was missing**, **what I wrote**, and **why it works that way**, so it can be used as study material.

---

## 1. What the project is

Holofix simulates remote maintenance assistance with smart glasses.

```
[glasses]  <--JSON-->  [app.py (API)]  <--JSON-->  [web panel]
                             |
                         [dados.py]
```

The glasses (`/oculos`) and the panel (`/painel`) are **already built** (`static/*.html`). They only talk to the server. So the whole task lives in **one file: `app.py`**.

`dados.py` is the in-memory "database" — plain Python dicts. When the server restarts, everything resets to the initial state. That's fine for a hackathon.

### The three data entities

| Entity | What it is | Stored in |
|---|---|---|
| `Maquina` | Equipment, keyed by the QR-code string (`CMP-01`) | `dados.maquinas` (dict) |
| `Procedimento` | An ordered list of steps | `dados.procedimentos` (dict) |
| `OrdemServico` | One maintenance job: technician + machine + current step | `dados.ordens` (dict) |
| `PedidoAjuda` | A help request from a technician to a specialist | `dados.pedidos_ajuda` (dict) |

---

## 2. What was missing

`app.py` shipped as a skeleton with **12 TODOs** across 5 missions. Every route returned `nao_implementado(N)` → HTTP 501. The result was: the pages loaded, but every button showed *"Mission N not implemented yet"*.

| Mission | Route(s) | Done? |
|---|---|---|
| 1. Machines | `GET /api/maquinas`, `GET /api/maquinas/<codigo>` | ✅ |
| 2. Service orders | `POST /api/ordens`, `GET /api/ordens/<id>` | ✅ |
| 3. Step navigation | `.../avancar`, `.../voltar`, `.../concluir` | ✅ |
| 4. Ask for help (bonus) | `POST /api/ordens/<id>/ajuda` | ✅ |
| 5. Specialist panel (bonus) | `GET /api/ajudas`, `.../responder`, `GET /api/ordens` | ✅ |

**Result: `python verificar.py` → 32/32 checks passed (minimum goal was 20/20).**

---

## 3. Mission by mission

### Mission 1 — Machines

**1.1 — list all machines**

```python
return jsonify(list(dados.maquinas.values()))
```

`dados.maquinas` is a *dict* keyed by code. JSON has no concept of a keyed dict here, so `.values()` pulls out the machine objects and `list(...)` turns them into a JSON array.

**1.2 — one machine by code**

```python
codigo = codigo.strip().upper()
if codigo not in dados.maquinas:
    return erro("Máquina não encontrada", 404)

maquina = dados.maquinas[codigo]
procedimento = dados.procedimentos[maquina["procedimento_id"]]

resposta = dict(maquina)          # copy!
resposta["procedimento"] = procedimento["titulo"]
resposta["total_passos"] = len(procedimento["passos"])
return jsonify(resposta)
```

Three ideas worth studying here:

1. **Normalising input.** The user may scan/type `cmp-01`. `.strip().upper()` makes it match the key `CMP-01`. Note this is the *only* place the code is normalised — everything else stores the already-normalised version.
2. **`in` before `[]`.** `dados.maquinas[codigo]` on a missing key raises `KeyError` → HTTP 500. Checking `in` first lets us return a clean **404**.
3. **`dict(maquina)` is a copy.** This is the important one. If I had written `maquina["procedimento"] = ...`, I would be writing the extra fields **into the database itself**, permanently polluting `dados.maquinas`. `dict(x)` creates a shallow copy, so the enrichment stays in the response only.

> **`KeyError` vs `.get()`** — two styles appear in this project:
> - `dados.maquinas[codigo]` → explodes if missing (used when we already validated)
> - `dados.ordens.get(id)` → returns `None` if missing (used when missing is a normal case)
>
> Both are correct; pick based on whether "missing" is a bug or an expected outcome.

---

### Mission 2 — Open and read a service order

**2.1–2.4 — create an order**

```python
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

Details:

- **`request.get_json(silent=True) or {}`** — the `silent=True` means "if the body isn't valid JSON, return `None` instead of raising". The `or {}` then makes it an empty dict, so `corpo.get(...)` never crashes. This is what lets a malformed request produce a tidy **400** instead of a **500**.
- **`(x or "")`** — `corpo.get("tecnico")` returns `None` when the key is absent. `None.strip()` would crash. `(None or "")` → `""` → `"".strip()` → `""`. Then `if not tecnico` catches it.
- **`passo_atual` starts at 0**, not 1. Steps are indexed `0..total-1`, which is why the UI everywhere shows `passo_atual + 1` to the human.
- **`201 Created`** vs `200 OK`. `201` means "a new resource now exists". The panel/glasses don't strictly need it, but `verificar.py` checks for it.
- **Validation order matters.** Missing fields → `400`; well-formed but pointing at a nonexistent machine → `404`. Sending `{"maquina_codigo": "NAO-EXISTE"}` with no `tecnico` returns `400`, because the `400` check comes first.

**2.5 — read an order**

```python
ordem = dados.ordens.get(ordem_id)
if ordem is None:
    return erro("Ordem não encontrada", 404)
return jsonify(montar_ordem(ordem))
```

The glasses call this **every 3 seconds** to poll for the specialist's answer (see `oculos.html`). That polling loop is the whole "real-time" mechanism — no websockets needed.

---

### Mission 3 — Step navigation

All three routes follow the same recipe, which is spelled out in the comment block above them:

```
1. fetch the order            -> None? 404
2. already concluded?         -> 400
3. mission-specific rule      -> not allowed? 400
4. mutate + return montar_ordem(ordem)
```

| Route | Rule |
|---|---|
| `avancar` | `passo_atual >= total - 1` → 400 |
| `voltar` | `passo_atual <= 0` → 400 |
| `concluir` | `passo_atual < total - 1` → 400 |

**Off-by-one, the classic trap.** The last step is `total_de_passos(ordem) - 1`, **not** `total_de_passos(ordem)`. With 4 steps (`IMP-03`), the valid positions are `0, 1, 2, 3`. Writing `>= total_de_passos(ordem)` would let the technician walk off the end into `IndexError` inside `montar_ordem`, which does `passos[ordem["passo_atual"]]`.

`concluir` sets two fields:

```python
ordem["status"] = "concluida"
ordem["fim"] = dados.agora()
```

And note the order of checks in `concluir`: the `"concluida"` check comes **before** the last-step check. Otherwise, calling `concluir` twice would pass the step check (the step never moves back) and re-stamp `fim` with a new time.

> **Why the "already concluded" check exists at all:** once an order is done, its steps are frozen history. Letting `avancar` mutate a concluded order would corrupt the record of what actually happened.

---

### Mission 4 — Ask the specialist (bonus)

```python
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
    "passo": ordem["passo_atual"],   # <-- where the technician is RIGHT NOW
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

Key design points:

- **`"passo"` snapshots the current step.** The specialist needs to know *where* the technician is stuck. Storing it (rather than reading it live later) means the request still makes sense even if the technician keeps navigating.
- **The order's status changes to `"aguardando_ajuda"`.** This is a **state machine**: `em_andamento` → `aguardando_ajuda` → `em_andamento` → `concluida`. The panel renders each state with a different colour badge.
- Note this route returns the **raw `pedido`**, not `montar_ordem(...)`. `montar_ordem` is for *orders*; this returns a *help request*. Matching the right shape to the right endpoint matters.

---

### Mission 5 — Specialist panel (bonus)

**5.1 — list help requests, with optional filter**

```python
filtro = request.args.get("status")   # None when no ?status=... in the URL

lista = []
for pedido in dados.pedidos_ajuda.values():
    if filtro and pedido["status"] != filtro:
        continue
    item = dict(pedido)
    item["ordem"] = montar_ordem(dados.ordens[pedido["ordem_id"]])
    lista.append(item)

return jsonify(lista)
```

- **`request.args.get("status")`** reads the *query string* (`?status=aberto`), as opposed to `request.get_json()` which reads the *body*. GET requests carry data in the URL; POST requests carry it in the body.
- **`if filtro and ...`** — `filtro` is `None` when absent, which is falsy, so no filtering happens. This is the idiomatic Python "optional filter" pattern.
- **The nested `"ordem"` is the point of this route.** The panel needs to display *who* asked and *on which machine* (`painel.html` reads `p.ordem.tecnico`, `p.ordem.maquina_codigo`). Without it the panel would show a bare message with no context.

**5.2–5.5 — answer a request**

```python
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

The `if ordem["status"] == "aguardando_ajuda"` guard is deliberate: it only un-blocks the order if it was actually blocked. If the technician had already moved on, we don't clobber their state.

**5.6 — list all orders for the panel's right-hand table**

```python
return jsonify([montar_ordem(o) for o in dados.ordens.values()])
```

A list comprehension: "for each order, run it through `montar_ordem`, collect the results."

---

## 4. The central piece: `montar_ordem`

This function was **already written** and never needs changing — but understanding it is the key to the whole project.

```python
def montar_ordem(ordem):
    maquina = dados.maquinas[ordem["maquina_codigo"]]
    procedimento = dados.procedimentos[maquina["procedimento_id"]]
    passos = procedimento["passos"]

    ultima_ajuda = None
    for pedido in dados.pedidos_ajuda.values():
        if pedido["ordem_id"] == ordem["id"]:
            ultima_ajuda = pedido      # keeps the most recent
    ...
```

It's a **join**, like SQL `JOIN`. The stored order only holds *IDs* (`maquina_codigo`). But the glasses need *display text* (`maquina_nome`, `procedimento`, `texto_passo`). `montar_ordem` resolves the IDs into readable text and flattens it into one object.

**This is why the glasses see the specialist's answer for free.** Look at the last line:

```python
"ajuda": ultima_ajuda,
```

That's a **reference** to the very same dict stored in `dados.pedidos_ajuda`. When `responder_ajuda` does `pedido["resposta"] = resposta`, it mutates that shared object. The next time the glasses poll `/api/ordens/<id>` (within 3 seconds), `montar_ordem` hands back the same dict — now containing the answer. No copying, no notification system.

The loop relies on **dict insertion order** (guaranteed in Python 3.7+): the last matching entry wins, and entries were inserted in creation order, so it picks the newest request.

---

## 5. Concepts to take away

**HTTP status codes used in this project**

| Code | Meaning | Used when |
|---|---|---|
| `200` | OK | Normal read/update |
| `201` | Created | A new order or help request was stored |
| `400` | Bad Request | The *client* sent something wrong (empty field, step out of range) |
| `404` | Not Found | The *resource* doesn't exist (bad code, bad ID) |
| `501` | Not Implemented | The skeleton's placeholder — all gone now |

The 400/404 distinction is worth internalising: **400 = "your request is malformed", 404 = "your request is fine, but that thing isn't here."**

**Mutate vs. copy**

| Operation | Effect |
|---|---|
| `dict(maquina)` | Copy — safe to add fields to the response |
| `ordem["passo_atual"] += 1` | Mutation — the change *persists*, because `ordem` is a reference into `dados.ordens` |
| `list(dados.maquinas.values())` | New list, but the *elements* are still references |

`dados.ordens.get(id)` returns a reference to the stored dict, **not** a copy. So `ordem["status"] = "concluida"` writes straight through to the "database". That's intentional — it's what makes persistence work with no save step.

**Input normalisation at the boundary**

User input is normalised **once**, at the edge (`.strip().upper()`), and stored already-normalised. Everything downstream — dict lookups, comparisons, the UI — can then assume clean data.

---

## 6. How to run and verify

```bash
cd holofix-main
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt

.venv/bin/python app.py          # start the server

# in another terminal:
.venv/bin/python verificar.py    # all missions
.venv/bin/python verificar.py 3  # just mission 3
```

Then open `http://127.0.0.1:5000/` for the three links.

> **macOS gotcha I hit:** macOS AirPlay Receiver also listens on port 5000 (via `ControlCenter`), on IPv6. `localhost` may resolve to `::1` and hit AirPlay instead of Flask, returning non-JSON. Use **`127.0.0.1:5000`** explicitly, or turn AirPlay Receiver off in System Settings. This is *not* a bug in the code.

**Manual test script** (server running) — walks the full technician ↔ specialist loop:

```bash
B=http://127.0.0.1:5000

# technician opens an order
curl -s -X POST $B/api/ordens -H 'Content-Type: application/json' \
  -d '{"maquina_codigo":"CMP-01","tecnico":"Ana"}'

# advance a step
curl -s -X POST $B/api/ordens/1/avancar

# technician asks for help
curl -s -X POST $B/api/ordens/1/ajuda -H 'Content-Type: application/json' \
  -d '{"mensagem":"Nao encontro a peca"}'

# specialist sees it
curl -s "$B/api/ajudas?status=aberto"

# specialist answers
curl -s -X POST $B/api/ajudas/1/responder -H 'Content-Type: application/json' \
  -d '{"resposta":"Olhe atras do painel lateral"}'

# glasses now see the answer
curl -s $B/api/ordens/1        # -> "ajuda": { "resposta": "Olhe atras...", "status": "respondido" }
```

---

## 7. Verified behaviour

`python verificar.py` output:

```
[COMPLETA]  Missão 1 - Máquinas: 6/6
[COMPLETA]  Missão 2 - Abrir e consultar ordem de serviço: 7/7
[COMPLETA]  Missão 3 - Navegar pelos passos: 7/7
[COMPLETA]  Missão 4 - Pedir ajuda (bônus): 5/5
[COMPLETA]  Missão 5 - Painel do especialista (bônus): 7/7

Total: 32/32 verificações
Meta mínima (missões 1 a 3): 20/20
```

Beyond the automated checks, I confirmed over real HTTP:

- `cmp-01` lowercase and `"  Ana  "` with whitespace are normalised correctly
- `voltar` at step 0 → `400`, `avancar` past the last step → `400`
- `concluir` on the last step sets `status = "concluida"` and stamps `fim`
- `concluir` a second time → `400 "Ordem já concluída"`
- The full help loop: request → order becomes `aguardando_ajuda` → answer → order returns to `em_andamento` → the glasses read the answer from `ordem["ajuda"]`
- `?status=aberto` correctly excludes answered requests
- Answering the same request twice → `400 "Pedido já respondido"`

---

## 8. Files touched

| File | Change |
|---|---|
| `app.py` | Implemented all 12 TODOs across the 5 missions |
| `.gitignore` | **New** — excludes `.venv/`, `__pycache__/`, `.DS_Store` |
| `RELATORIO.md` | **New** — this report |

`dados.py`, `verificar.py`, and everything in `static/` were left untouched, as the README requires.

---

## 9. Optional next steps

The skeleton mentions "extra challenges" with no TODOs attached. Natural extensions, in rough order of difficulty:

1. **Persistence** — dump `ordens` / `pedidos_ajuda` to a JSON file or SQLite so a restart doesn't wipe everything.
2. **Concurrency** — `dados.ordens` is a plain dict mutated by multiple request threads. Flask's dev server is threaded by default; a `threading.Lock` around mutations would be the correct fix.
3. **Order history per machine** — an endpoint returning all past orders for a given `maquina_codigo`.
4. **Timing metrics** — using `inicio`/`fim`, compute how long each maintenance took.
