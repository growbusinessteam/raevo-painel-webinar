#!/usr/bin/env python3
"""Gera data.json do painel do webinar e publica no GitHub se algo mudou.

Fontes: quadro WEBINAR do CRM Raevo, tarefas do ClickUp (config.json) e o estado do
aquecimento automático. As chaves vêm do Chaveiro do Mac (raevo-crm-api, clickup-api).

  python3 tools/atualizar.py              # gera e publica
  python3 tools/atualizar.py --sem-push   # só gera data.json
  python3 tools/atualizar.py --docs       # também copia os documentos finais (não corre no launchd)
"""
import datetime as dt, json, os, shutil, subprocess, sys, urllib.request
from zoneinfo import ZoneInfo

RAIZ = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
CFG = json.load(open(os.path.join(RAIZ, "config.json")))
TZ = ZoneInfo(CFG["aula"]["fuso"])


def chave(servico):
    return subprocess.run(["security", "find-generic-password", "-s", servico, "-a", "token", "-w"],
                          capture_output=True, text=True, check=True).stdout.strip()


def get(url, cab):
    with urllib.request.urlopen(urllib.request.Request(url, headers=cab), timeout=60) as r:
        return json.load(r)


def funil():
    d = get(f"https://crm.raevo.io/api/v1/accounts/1/kanban_boards/{CFG['crm_quadro_webinar']}",
            {"api_access_token": chave("raevo-crm-api")})
    b = d.get("payload", d)
    return [{"etapa": s["name"], "total": s.get("cards_count", len(s.get("cards") or []))}
            for s in b.get("stages", [])]


def tarefas():
    cab = {"Authorization": chave("clickup-api")}
    vistos, saida = set(), []

    def item(t, pai=None):
        if t["id"] in vistos:
            return
        vistos.add(t["id"])
        st = t.get("status") or {}
        tipo = st.get("type", "open")
        estado = "feito" if tipo in ("done", "closed") else ("pendente" if tipo == "open" else "em curso")
        prazo = t.get("due_date")
        saida.append({
            "id": t["id"], "nome": t["name"], "url": t.get("url") or f"https://app.clickup.com/t/{t['id']}",
            "estado": estado, "status": st.get("status", ""), "cor": st.get("color", ""),
            "responsaveis": [a.get("username", "") for a in t.get("assignees", [])],
            "prazo": dt.datetime.fromtimestamp(int(prazo) / 1000, TZ).date().isoformat() if prazo else None,
            "lista": (t.get("list") or {}).get("name"), "pai": pai,
            "atualizado": dt.datetime.fromtimestamp(int(t["date_updated"]) / 1000, TZ).isoformat() if t.get("date_updated") else None,
        })

    for tid in CFG["clickup_tarefas"]:
        try:
            t = get(f"https://api.clickup.com/api/v2/task/{tid}?include_subtasks=true", cab)
        except Exception as e:
            print("ClickUp falhou", tid, e, file=sys.stderr)
            continue
        item(t)
        for s in t.get("subtasks") or []:
            s.setdefault("list", t.get("list"))
            item(s, pai=t["name"])
    return saida


def aquecimento():
    """Mensagens dos últimos 3 dias e dos próximos 7, com o estado de cada envio."""
    pasta = CFG["aquecimento_pasta"]
    seq = json.load(open(os.path.join(pasta, "sequencia.json")))
    estado = json.load(open(os.path.join(pasta, "estado.json"))) if os.path.exists(os.path.join(pasta, "estado.json")) else {}
    log = open(os.path.join(pasta, "envios.log")).read().splitlines() if os.path.exists(os.path.join(pasta, "envios.log")) else []
    agora = dt.datetime.now(TZ)
    seg_atual = (agora - dt.timedelta(days=agora.weekday())).date()
    msgs = []
    for semanas in (-1, 0, 1):
        segunda = seg_atual + dt.timedelta(weeks=semanas)
        fim = (segunda + dt.timedelta(days=7)).isoformat()
        semana = [l for l in log if segunda.isoformat() <= l[:10] < fim]
        feitos = estado.get(segunda.isoformat(), [])
        for m in seq["mensagens"]:
            h, mi = map(int, m["hora"].split(":"))
            quando = dt.datetime.combine(segunda + dt.timedelta(days=m["dia"]), dt.time(h, mi), tzinfo=TZ)
            if not (agora - dt.timedelta(days=3) <= quando <= agora + dt.timedelta(days=7)):
                continue
            if any(f"SALTADA {m['id']}" in l for l in semana):
                st = "saltada"
            elif any(f" {m['id']}:" in l and ("ENVIADA" in l or "TESTE" in l) or f"TESTE {m['id']} " in l for l in semana):
                st = "enviada"
            elif m["id"] in feitos or quando < agora - dt.timedelta(minutes=30):
                st = "não saiu"
            else:
                st = "agendada"
            msgs.append({"id": m["id"], "quando": quando.isoformat(), "estado": st, "texto": m["texto"],
                         "imagens": len(m.get("imagens", []))})
    msgs.sort(key=lambda x: x["quando"])
    return {"pausado": os.path.exists(os.path.join(pasta, "PAUSADO")), "mensagens": msgs,
            "ultimos_registos": log[-6:]}


def timer():
    """Tarefa com o timer do ClickUp a correr agora (do dono da chave), se houver."""
    cab = {"Authorization": chave("clickup-api")}
    equipa = get("https://api.clickup.com/api/v2/team", cab)["teams"][0]["id"]
    d = (get(f"https://api.clickup.com/api/v2/team/{equipa}/time_entries/current", cab) or {}).get("data")
    if not d or not d.get("task"):
        return None
    return {"id": d["task"]["id"], "nome": d["task"]["name"],
            "desde": dt.datetime.fromtimestamp(int(d["start"]) / 1000, TZ).isoformat(timespec="minutes")}


def convites():
    """Totais (sem dados pessoais) dos convites da rede para a aula."""
    pasta = os.path.expanduser("~/Library/Application Support/raevo-convites-rede")
    lista = json.load(open(os.path.join(pasta, "lista.json")))
    est = json.load(open(os.path.join(pasta, "estado.json"))) if os.path.exists(os.path.join(pasta, "estado.json")) else {"leads": {}}
    L = est.get("leads", {})
    env = [e for e in L.values() if e.get("enviado_ts")]
    conta = lambda f: sum(1 for e in env if e.get("fim") == f)
    grupos = {}
    for x in lista:
        g = grupos.setdefault(x["grupo"], {"total": 0, "enviados": 0, "sim": 0, "responderam": 0})
        g["total"] += 1; e = L.get(x["telefone"], {})
        if e.get("enviado_ts"): g["enviados"] += 1
        if e.get("fim") in ("sim", "nao", "respondeu"): g["responderam"] += 1
        if e.get("fim") == "sim": g["sim"] += 1
    return {"lista": len(lista), "enviados": len(env), "sim": conta("sim"), "nao": conta("nao"),
            "outras_respostas": conta("respondeu"), "sem_resposta": conta("sem-resposta"),
            "a_aguardar": sum(1 for e in env if e.get("fim") in (None, "lembrado")), "grupos": grupos}


def proxima_aula():
    agora = dt.datetime.now(TZ)
    h, mi = map(int, CFG["aula"]["hora"].split(":"))
    dias = (CFG["aula"]["dia_semana"] - agora.weekday()) % 7
    aula = dt.datetime.combine(agora.date() + dt.timedelta(days=dias), dt.time(h, mi), tzinfo=TZ)
    if aula + dt.timedelta(hours=1) < agora:
        aula += dt.timedelta(days=7)
    return aula.isoformat()


def copiar_docs():
    for d in CFG["documentos"]:
        destino = os.path.join(RAIZ, d["destino"])
        os.makedirs(os.path.dirname(destino), exist_ok=True)
        shutil.copy2(d["origem"], destino)


def main():
    if "--docs" in sys.argv:
        copiar_docs()
    dados = {"gerado_em": dt.datetime.now(TZ).isoformat(timespec="minutes"),
             "aula": {"proxima": proxima_aula(), "nota": CFG["aula"]["nota"]},
             "links": CFG["links"],
             "documentos": [{"nome": d["nome"], "caminho": d["destino"]} for d in CFG["documentos"]],
             "erros": []}
    for nome, fn in (("funil", funil), ("tarefas", tarefas), ("aquecimento", aquecimento), ("timer", timer), ("convites", convites)):
        try:
            dados[nome] = fn()
        except Exception as e:
            dados[nome] = None
            dados["erros"].append(f"{nome}: {e}")
    caminho = os.path.join(RAIZ, "data.json")
    antigo = json.load(open(caminho)) if os.path.exists(caminho) else {}
    novo_sem_hora = {k: v for k, v in dados.items() if k != "gerado_em"}
    antigo_sem_hora = {k: v for k, v in antigo.items() if k != "gerado_em"}
    json.dump(dados, open(caminho, "w"), ensure_ascii=False, indent=1)
    if "--sem-push" in sys.argv:
        return
    git = lambda *a: subprocess.run(["git", "-C", RAIZ, *a], capture_output=True, text=True)
    # publica quando os dados mudam, ou a cada 6 h para mostrar que o painel está vivo
    velho = antigo.get("gerado_em")
    atrasado = not velho or (dt.datetime.now(TZ) - dt.datetime.fromisoformat(velho)).total_seconds() > 6 * 3600
    outros = [l for l in git("status", "--porcelain").stdout.splitlines() if not l.endswith("data.json")]
    if novo_sem_hora == antigo_sem_hora and not atrasado and not outros:
        git("checkout", "--", "data.json")
        return
    if True:
        git("add", "-A")
        git("commit", "-m", f"Atualiza dados {dados['gerado_em']}")
        r = git("push", "-q")
        if r.returncode:
            print("push falhou:", r.stderr, file=sys.stderr)


if __name__ == "__main__":
    main()
