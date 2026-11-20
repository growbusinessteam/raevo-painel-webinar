# Painel do webinar Raevo

Página: https://growbusinessteam.github.io/raevo-painel-webinar/

Painel aberto (sem login) do webinar "Atendimento Organizado": próxima aula, o que falta fazer, funil do CRM, aquecimento da semana, links e documentos finais. Não guardar aqui dados de leads (nomes, telefones) nem chaves.

## Como se atualiza

- `tools/atualizar.py` gera `data.json` a partir de:
  - **CRM Raevo**: quadro WEBINAR (id 2), contagem por etapa. Chave no Chaveiro: `raevo-crm-api`.
  - **ClickUp**: tarefas listadas em `config.json` → `clickup_tarefas`, mais as subtarefas delas. Chave no Chaveiro: `clickup-api`.
  - **Aquecimento automático**: estado em `~/Library/Application Support/raevo-webinar-aquecimento/`.
- O Mac do Alysson corre o script a cada hora (`~/Library/LaunchAgents/io.raevo.painel-webinar.plist`) e faz push quando os dados mudam. Com o Mac desligado, o painel fica com os últimos dados.
- A cópia local do repositório fica em `~/Library/Application Support/raevo-painel-webinar/` (fora de Documentos, porque o agendador do macOS não lê essa pasta).

## Para mudar o painel (Claude, Codex ou equipa)

- **Nova tarefa do ClickUp no painel:** acrescentar o ID em `config.json` → `clickup_tarefas` (subtarefas entram sozinhas).
- **Novo link:** `config.json` → `links`.
- **Novo documento final:** `config.json` → `documentos` (origem no Mac + destino em `docs/`), depois correr `python3 tools/atualizar.py --docs` a partir de um terminal (o agendador não copia documentos).
- **Prioridades e próximos passos:** `foco.json` (ordem, quem, próximo passo, prazo proposto, o que o Claude faz sozinho, tarefas a propor fechar). Rever depois de cada conversa de trabalho.
- **Layout:** `index.html` (lê `data.json`); documentos `.md` abrem em `ver.html`.
