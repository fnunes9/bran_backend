# Bran — agente de QA de backend

O Bran é uma **skill do Claude Code**: um manual de comportamento que transforma o Claude num QA de backend. Ele lê a tarefa do kanban, cruza com a documentação, monta e executa as chamadas de API, confere o banco de dados, registra o resultado na tarefa e avisa no Discord.

> **Skill**, em poucas palavras: um arquivo de instruções (`SKILL.md`) que o Claude carrega quando você chama pelo nome (`/bran`). Não é um programa: é o "jeito de trabalhar" do agente. Os scripts da pasta `scripts/` são as ferramentas que ele usa.

---

## 1. Mapa da pasta

```
Bran/
├── CLAUDE.md                   regras desta pasta (o Bran é independente; limites)
├── README.md                   este guia
├── .env.example                modelo das variáveis (sem valores; pode compartilhar)
├── .env                        seus valores reais (NUNCA compartilhar)
├── .gitignore                  impede .env e .cache/ de irem para o Git
├── .claude/
│   ├── skills/bran/SKILL.md    a skill: como o Bran testa e decide
│   ├── settings.json           liga o hook de proteção do banco
│   └── hooks/guard_sql.sh      bloqueia DELETE/UPDATE sem WHERE e DROP/TRUNCATE/ALTER
├── config/
│   ├── perfil.md               seu perfil: kanban, colunas, classificações, sistemas
│   └── login.sh                como fazer login na sua API (imprime só o token)
├── scripts/
│   ├── api_request.sh          chamadas de API com token automático
│   ├── get_token.sh            token com cache (sem login a toda hora)
│   ├── db_query.sh             consultas ao MySQL
│   ├── discord_send.sh         mensagens ao Discord
│   └── _load_env.sh            carrega o .env (usado pelos outros scripts)
├── api/                        collection do Bruno (criada por você no Bruno)
└── Epicos/
    └── <epico>/
        ├── documentacao.md     documentação de produto
        ├── duvidas.md          doc de dúvidas (vence as outras fontes)
        ├── regras-aprendidas/regras.md
        └── <TAREFA>/           relatorio_AAAA-MM-DD.md + evidencias/
```

**Como as partes se conectam:** você chama `/bran BMD-1234` → o Claude carrega a `SKILL.md` → lê `config/perfil.md` para saber como falar com o kanban → lê a tarefa e os documentos do épico → executa os testes com os scripts (que leem o `.env`) → grava o relatório em `Epicos/` e atualiza a tarefa no kanban.

---

## 2. Pré-requisitos

| O quê | Para quê | Como conferir |
|---|---|---|
| Claude Code (terminal ou extensão do VS Code) | Rodar o Bran | `claude --version` |
| `curl` e `jq` | Chamadas de API e leitura de JSON | `curl --version` e `jq --version` |
| Cliente MySQL de terminal | Consultas ao banco | `mysql --version` |
| Bruno (aplicativo) | Ver e repetir as requisições | Abrir o aplicativo |
| Acesso ao kanban com integração MCP | Ler e atualizar tarefas | `claude mcp list` |
| Usuário de teste **exclusivo** do Bran na API de homolog | Login sem derrubar a sessão de ninguém | — |
| Acesso de leitura ao banco de homolog | Conferir o que a API gravou | — |
| Webhooks do Discord (reprovação e produto) | Avisos do Bran | — |

> **MCP**, em poucas palavras: um padrão de "tomada" que dá ao Claude ferramentas novas. O kanban expõe um servidor MCP; ao registrá-lo, o Claude ganha ferramentas como "buscar tarefa" e "comentar".

---

## 3. Instalação passo a passo

Os comandos são para Linux (Ubuntu/Debian). Rode na raiz da pasta do Bran.

### 3.1. Ferramentas básicas
```bash
sudo apt update
sudo apt install curl jq mysql-client
```
- `apt update` atualiza a lista de pacotes; `apt install` instala. Vai pedir sua senha do Linux.
- O pacote `mysql-client` instala **só o cliente** (não um servidor). Se aparecer "pacote não encontrado", tente `default-mysql-client`.
- Conferir: `mysql --version` deve mostrar algo como `mysql  Ver 8.x`.

### 3.2. Scripts executáveis
```bash
chmod +x scripts/*.sh config/login.sh .claude/hooks/guard_sql.sh
```
`chmod +x` dá permissão de execução. Conferir com `ls -l scripts/`: as linhas devem começar com `-rwx`.

### 3.3. Kanban
Primeiro descubra se o seu kanban tem servidor MCP (procure "MCP" na documentação da ferramenta). Se não souber como integrar, abra o Claude Code nesta pasta e rode `/bran setup`: o Bran faz perguntas sobre o seu kanban e te guia.

**Exemplo com servidor MCP HTTP e token** (o caso do Maestro):
```bash
read -s KANBAN_TOKEN
claude mcp add --transport http --scope local <nome> <URL_DO_SERVIDOR_MCP> --header "Authorization: Bearer $KANBAN_TOKEN"
claude mcp list
```
- `read -s` guarda o token numa variável sem mostrar na tela nem gravar no histórico do terminal. Cole o token e aperte Enter (não aparece nada: é normal).
- `--scope local` grava a configuração só na sua máquina: o token não vai para arquivos compartilháveis.
- `claude mcp list` deve mostrar `<nome>: <URL> (HTTP) - ✔ Connected`.
- Se o servidor usar login pelo navegador (OAuth), registre sem `--header` e, dentro do Claude Code, rode `/mcp` para entrar.
- **Depois de conectar, abra uma conversa nova**: as ferramentas carregam no início da sessão.

Em seguida, preencha `config/perfil.md` (colunas, classificações, nomes das ferramentas). O `/bran setup` ajuda.

### 3.4. Arquivo `.env`
```bash
cp .env.example .env
chmod 600 .env
```
Abra o `.env` e preencha. `chmod 600` deixa o arquivo legível só por você. Valores com espaço ou caractere especial vão entre aspas: `DB_PASSWORD="a#b c"`.

**Onde encontrar os dados do banco:** no seu cliente gráfico (ex.: Beekeeper Studio), abra a conexão de homolog e copie host, porta, usuário e nome do banco.

Conferir o banco:
```bash
scripts/db_query.sh "SELECT 1;"
```
Esperado: uma tabelinha com o valor `1`. Erro `Access denied` = usuário ou senha errados; `Can't connect` = host, porta ou VPN.

### 3.5. Login da API (`config/login.sh`)
Cada API faz login de um jeito, então este arquivo é personalizado. Pegue o `curl` de login (do Bruno, da documentação ou do time), **troque a senha por `SENHA_AQUI`** e entregue ao Bran: ele monta o `login.sh` usando as variáveis do `.env`.

Conferir:
```bash
scripts/get_token.sh > /dev/null && echo "login ok"
```
Esperado: `login ok`. O `> /dev/null` descarta o token, para ele não aparecer na tela.

> **Por que um cache de token?** Fazer login a cada chamada é lento e pode derrubar sessões. O `get_token.sh` guarda o token em `.cache/token` e só faz login de novo quando ele vence. Se o token for um **JWT** (texto com três partes separadas por pontos), a validade é lida de dentro dele (campo `exp`); senão, vale `TOKEN_TTL_SECONDS`.

### 3.6. Collection do Bruno
> **Collection**, em poucas palavras: uma pasta de requisições salvas. No Bruno, cada requisição é um arquivo de texto, então a collection pode ficar aqui dentro e ser lida pelo Bran.

1. Abra o Bruno → **Create Collection**.
2. Nome: `api`. Local: a pasta raiz do Bran.
3. Crie as pastas por épico ou recurso e salve as requisições que você já conhece.

O Bran procura primeiro na collection, depois nos documentos, e **pergunta** o que faltar: ele não inventa rota, método nem campo. Requisições novas que ele criar aparecem no Bruno para você repetir o teste.

### 3.7. Épico
Para cada épico, crie a pasta `Epicos/<nome_do_epico>/` com:
- `documentacao.md`: documentação de produto (Google Docs: Arquivo → Fazer download → Markdown);
- `duvidas.md`: doc de dúvidas do time.

Se você esquecer, o Bran pede na primeira tarefa do épico.

### 3.8. Conferência final
Abra o Claude Code na pasta do Bran e rode:
```
/bran setup
```
O Bran confere tudo e diz o que falta.

---

## 4. Como usar

| Comando | O que acontece |
|---|---|
| `/bran BMD-1234` | Testa a tarefa: planejamento com você → execução sozinho → resultado no kanban |
| `/bran BMD-1234` (tarefa já testada) | Reteste: responde às revisões abertas e retesta o que mudou |
| `/bran setup` | Confere os pré-requisitos e guia o que faltar |
| `Bran, aprenda com isso` | Grava uma regra em `regras-aprendidas/regras.md` do épico |

**O fluxo:**
1. **Planejamento (com você):** o Bran lê a tarefa e os documentos, mapeia endpoints e tabelas, pede o que faltar e mostra o plano de teste. Você dá o ok.
2. **Execução (sozinho):** chama a API, confere o banco, salva as evidências.
3. **Resultado:** uma revisão por caso de teste, comentário geral, tarefa movida, Discord (se reprovar) e relatório em `Epicos/<epico>/<TAREFA>/`.

**Prioridade das fontes:** doc de dúvidas > documentação de produto > tarefa. O que nenhuma fonte resolve vira dúvida no canal de produto do Discord.

---

## 5. Comandos úteis (você também pode usar)

```bash
# Chamada de API (token automático)
scripts/api_request.sh GET /rota
scripts/api_request.sh POST /rota corpo.json
scripts/api_request.sh --sem-token GET /rota

# Banco (SQL sempre entre aspas)
scripts/db_query.sh "SELECT * FROM tabela WHERE id = 1;"

# Forçar novo login
scripts/get_token.sh --force > /dev/null && echo ok

# Discord
scripts/discord_send.sh produto mensagem.txt
```

---

## 6. Segurança

- **Somente homolog.** Nunca aponte o `.env` para produção.
- **`.env` nunca é compartilhado** nem vai para o Git (já está no `.gitignore`). Para compartilhar o kit, mande o `.env.example`.
- **Hook de proteção do banco:** antes de qualquer comando, o Claude Code roda `.claude/hooks/guard_sql.sh`, que bloqueia `DELETE`/`UPDATE` sem `WHERE` e `DROP`/`TRUNCATE`/`ALTER`. Limitação: ele lê só o texto do comando, por isso o Bran sempre passa o SQL inline no `db_query.sh`.
- **Usuário de teste exclusivo:** o Bran usa só o usuário do `.env`, para não encerrar a sessão de ninguém.

---

## 7. Problemas comuns

| Sintoma | Onde olhar |
|---|---|
| `/bran` não aparece | A pasta aberta no Claude Code é a raiz do Bran? Existe `.claude/skills/bran/SKILL.md`? |
| Ferramentas do kanban não aparecem | `claude mcp list` mostra Connected? Abriu uma conversa nova depois de conectar? |
| `ERRO: variáveis vazias no .env: ...` | Preencha as variáveis listadas no `.env` |
| `ERRO: config/login.sh ainda não foi configurado` | Seção 3.5 |
| Todas as chamadas dão 401 | Rode `scripts/get_token.sh --force > /dev/null`; se continuar, confira `AUTH_HEADER_NAME`/`AUTH_SCHEME` no `.env` e o `login.sh` |
| `BLOQUEADO pelo hook de proteção do banco` | É proposital: o SQL viola uma regra. Revise o SQL |
| `Permission denied` ao rodar script | Seção 3.2 |
