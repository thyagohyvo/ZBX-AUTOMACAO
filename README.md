# Zabbix Automation Panel

Painel web para automatizar tarefas operacionais recorrentes de um ambiente **Zabbix**: criação de dependências de trigger, monitoramento de housekeeping, importação em massa de hosts via CSV, gestão de janelas de manutenção, identificação de itens sem coleta de dados e ranking de hosts mais problemáticos.

Backend em **Flask (Python)** consumindo a **Zabbix API (JSON-RPC)**, frontend em **HTML/CSS/JavaScript puro** (sem frameworks).

---

## ✨ Funcionalidades

| Módulo | Descrição |
|---|---|
| **API Status** | Exibe o equivalente ao painel "System Information" do Zabbix: versão da API, hosts, templates, itens, triggers, usuários online e performance requerida (NVPS). |
| **Dependências** | Cria automaticamente dependências de trigger entre um host "parent" e todos os hosts de um grupo, com base no item de ping (`icmpping`) ou em regex de fallback. |
| **Housekeeping** | Coleta métricas do processo housekeeper (fila de exclusão, % de utilização) e configurações globais de retenção, gerando insights e recomendações automáticas. Inclui checagem específica de **auditoria (Audit Log)**: se o housekeeping do audit log está habilitado (`hk_audit_mode`) e se o período de retenção configurado (`hk_audit`) está excessivo (> 365 dias), alertando quando isso pode inflar o banco de dados sem ganho operacional. |
| **Importar Hosts** | Upload de CSV com preview/validação linha a linha (hostname, IP, grupo, templates, SNMP v1/v2/v3, tags, macros) antes da criação em massa via API. |
| **Manutenção** | Criação, listagem (ativas/agendadas/expiradas) e encerramento de janelas de manutenção por host ou grupo. |
| **Itens sem Dados** | Lista itens `not supported` ou sem coleta há mais de N horas, com filtro por grupo. |
| **Top Problemas** | Ranking de hosts por volume de eventos de trigger em um período (24h/7d/30d), com problemas ativos e top triggers por host. |
| **Histórico de Execuções** | Todas as execuções de dependências geram um log em arquivo `.txt`, listado e visualizável pela interface. |

### Checagens realizadas pelo módulo de Housekeeping

O backend (`analyze_housekeeping` em `app.py`) avalia, além da fila e do uso do processo housekeeper, se cada um dos seguintes módulos está com o housekeeping **habilitado** e se a **retenção** configurada é razoável:

- Histórico (`hk_history_mode` / `hk_history`)
- Trends (`hk_trends_mode` / `hk_trends`)
- Eventos (`hk_events_mode` / `hk_events_trigger`)
- **Auditoria / Audit log (`hk_audit_mode` / `hk_audit`)** — alerta se o housekeeping do audit log estiver desabilitado, e se a retenção configurada ultrapassar 365 dias
- Sessões de usuário (`hk_sessions_mode`)
- Serviços / SLA (`hk_services_mode`)
- Override global de histórico e de trends (`hk_history_global` / `hk_trends_global`)
- Compressão do TimescaleDB, quando aplicável

Cada verificação gera um item na lista de **Insights & Melhorias**, classificado por severidade (`critical`, `warning`, `improvement`), com descrição do problema e ação recomendada — visíveis na aba **Housekeeping** da interface.

---

## 🛠️ Stack

- **Backend:** Python 3 + [Flask](https://flask.palletsprojects.com/)
- **Integração:** Zabbix API (JSON-RPC via `requests`)
- **Frontend:** HTML5, CSS3 (dark theme customizado) e JavaScript vanilla
- **Persistência local:** arquivos `.txt` em `logs/` (histórico de execuções)

---

## 📁 Estrutura do projeto

```
.
├── app.py               # Backend Flask — rotas, integração com Zabbix API e regras de negócio
├── templates/
│   └── index.html       # Interface (SPA renderizada por troca de <div> visível)
├── static/
│   └── js/
│       └── app.js       # Lógica de frontend (fetch às rotas /api/*, renderização)
└── logs/                # Gerado automaticamente — logs das execuções de dependências
```

> ⚠️ O `index.html` referencia `/static/js/app.js`, então mantenha `app.js` dentro de `static/js/` e o `index.html` dentro de `templates/`, conforme a convenção do Flask.

---

## ⚙️ Configuração

As credenciais da API do Zabbix estão definidas no topo de `app.py`:

```python
ZABBIX_URL = "http://SEU_SERVIDOR:8080/api_jsonrpc.php"
ZABBIX_TOKEN = "SEU_TOKEN_DE_API"
```

**Recomendação de segurança:** não deixe URL e token hardcoded no código-fonte versionado. Substitua por variáveis de ambiente antes de subir para produção ou para um repositório público, por exemplo:

```python
import os

ZABBIX_URL = os.environ.get("ZABBIX_URL")
ZABBIX_TOKEN = os.environ.get("ZABBIX_TOKEN")
```

E defina no ambiente (ou em um arquivo `.env` com `python-dotenv`):

```bash
export ZABBIX_URL="http://seu-servidor:8080/api_jsonrpc.php"
export ZABBIX_TOKEN="seu-token-aqui"
```

O token precisa ter permissão de leitura/escrita nos módulos utilizados: `host`, `hostgroup`, `template`, `item`, `trigger`, `event`, `problem`, `maintenance`, `user`, `settings` e `housekeeping` (Zabbix 6.0+ para `housekeeping.get`).

---

## ▶️ Como rodar

```bash
# 1. Clone o repositório
git clone <url-do-repo>
cd <nome-do-repo>

# 2. Crie um ambiente virtual (opcional, recomendado)
python3 -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\activate

# 3. Instale as dependências
pip install flask requests

# 4. Configure ZABBIX_URL e ZABBIX_TOKEN (ver seção Configuração)

# 5. Execute
python app.py
```

A aplicação sobe em `http://0.0.0.0:5000`.

---

## 🔌 Principais rotas da API

| Método | Rota | Descrição |
|---|---|---|
| `GET` | `/api/test` | Testa conexão e retorna versão da API Zabbix |
| `GET` | `/api/sysinfo` | Resumo tipo "System Information" |
| `GET` | `/api/hosts` / `/api/groups` | Lista hosts / grupos |
| `GET` | `/api/housekeeping` | Métricas e análise de housekeeping |
| `GET` | `/api/logs` · `/api/logs/<file>` · `/api/logs/<file>/download` | Histórico de logs de execução |
| `POST` | `/api/dependency` | Executa criação de dependências de trigger |
| `POST` | `/api/import/preview` | Valida CSV de importação (sem gravar) |
| `POST` | `/api/import/run` | Executa a importação de hosts |
| `GET/POST` | `/api/maintenance` | Lista / cria janelas de manutenção |
| `DELETE` | `/api/maintenance/<id>` | Encerra uma manutenção |
| `GET` | `/api/nodata` | Itens sem dados / not supported |
| `GET` | `/api/topproblems` | Ranking de hosts por eventos de problema |

---

## 📌 Formato do CSV de importação

```
hostname,ip,group,template,community,port,snmp_version,tags,macros
sw-core-01,10.0.0.10,Switches,ICMP-SW-Base,public,161,2,site:SP,
olt-01,10.0.0.20,OLTs,ICMP-OLT-Base|SNMP-OLT,monitor,161,2,site:RJ,LOCATION:datacenter1
```

- `hostname`, `ip`, `group` e `template` são obrigatórios.
- Múltiplos templates são separados por `|`.
- `tags` e `macros` usam o formato `chave:valor`, separados por vírgula.
- Um modelo de CSV pode ser baixado diretamente pela interface (botão "Baixar modelo CSV").

---

## 📝 Licença

Defina aqui a licença do projeto (ex: MIT).
