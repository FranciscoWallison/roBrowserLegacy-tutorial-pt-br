#  Guia de Implantação: Ragnarok Online Web (rAthena + roBrowser)

Este guia detalha a criação de um ambiente Ragnarok completo rodando no navegador, composto por três pilares:

1. **Backend:** Servidor rAthena (Docker).
2. **Asset Server:** Servidor de arquivos (Remote Client).
3. **Frontend:** Interface do jogo (roBrowser Legacy).

---

## 🛠️ Parte 1: Pré-requisitos

Ferramentas essenciais para o funcionamento do ambiente.

### 1. Node.js (Runtime JavaScript)

Necessário para executar o cliente web e o proxy.

* **Download:** [nodejs.org](https://nodejs.org/en/download) (Recomendado: Versão LTS).
* **Validação:** No terminal, execute:
```powershell
node -v
npm -v

```


* **Instalação do Proxy Global:**
```powershell
npm install -g wsproxy

```



### 2. WSL 2 e Docker (Servidor)

Se já possui o Docker configurado, pule para a **Parte 2**.

1. **Verificar Arquitetura do Processador:**
```powershell
if ($env:PROCESSOR_ARCHITECTURE -eq "ARM64") { Write-Host "Baixe Docker ARM" } else { Write-Host "Baixe Docker x86_64" }

```


2. **Instalar WSL 2:**
Execute como Admin: `wsl --install` e **reinicie o computador**.
3. **Instalar Docker Desktop:**
Durante a instalação, marque a opção: `Use WSL 2 instead of Hyper-V`.

---

## 🖥️ Parte 2: Backend (rAthena via Docker)

Preparação do emulador para aceitar conexões do navegador.

### 1. Desativar Criptografia de Pacotes

O roBrowser precisa ler os pacotes "limpos". Edite o arquivo:
`rathena\src\custom\defines_post.hpp`

```cpp
#ifndef CONFIG_CUSTOM_DEFINES_POST_HPP
#define CONFIG_CUSTOM_DEFINES_POST_HPP

// Desativa a ofuscação para compatibilidade com roBrowser
#ifdef PACKET_OBFUSCATION
	#undef PACKET_OBFUSCATION
#endif
#ifdef PACKET_OBFUSCATION_WARN
	#undef PACKET_OBFUSCATION_WARN
#endif

#endif /* CONFIG_CUSTOM_DEFINES_POST_HPP */

```

### 2. Definir Versão do Cliente (PacketVer)

Vamos fixar a versão **20130618** (a mais estável para Web).
Edite o arquivo: `rathena\tools\docker\docker-compose.yml`

```yaml
# Localize a seção 'environment' e ajuste:
BUILDER_CONFIGURE: "--enable-packetver=20130618"

```

### 3. Compilar e Iniciar

Abra o PowerShell na pasta do Docker (`cd rathena\tools\docker`) e execute:

```powershell
# 1. Limpeza de containers e orfãos
docker compose down --remove-orphans

# 2. Remover binários antigos (garante recompilação limpa)
docker compose run --rm builder sh -c "rm -f /rathena/login-server /rathena/char-server /rathena/map-server /rathena/web-server"

# 3. Compilar o Servidor
docker compose run --rm builder

# 4. Iniciar (Login, Char e Map)
docker compose up -d db login char map

```

---

## 📂 Parte 3: Remote Client (Arquivos de Recursos)

Este projeto serve os arquivos `.grf`, músicas e dados do jogo para o navegador.

### 1. Instalação

1. Baixe ou clone: [roBrowserLegacy-RemoteClient-JS](https://github.com/FranciscoWallison/roBrowserLegacy-RemoteClient-JS)
2. Instale as dependências:
```powershell
cd roBrowserLegacy-RemoteClient-JS
npm install

```



### 2. Organização dos Arquivos (Assets)

Você deve copiar os arquivos do seu cliente Ragnarok para dentro da pasta deste projeto. A estrutura deve ficar assim:

```text
roBrowserLegacy-RemoteClient-JS/
├── AI/            <-- Copie do seu cliente
├── BGM/           <-- Copie do seu cliente
├── System/        <-- Copie do seu cliente
├── resources/
│   ├── data.grf   <-- Copie do seu cliente
│   └── DATA.INI   <-- Crie/Edite este arquivo

```

**Conteúdo do arquivo `resources\DATA.INI`:**

```ini
[Data]
0=data.grf

```

### 3. Iniciar Servidor de Arquivos

```powershell
npm start

```

*(Mantenha este terminal aberto)*

---

## 🌐 Parte 4: Frontend (O Navegador)

A interface que o jogador irá acessar.

### 1. Instalação e Configuração

1. Baixe ou clone: [roBrowserLegacy](https://github.com/MrAntares/roBrowserLegacy)
2. **Configurar Versão do Pacote:**
Vá até `roBrowserLegacy\applications\pwa\` e edite (ou crie) o arquivo `Config.local.js`. Adicione/altere a linha para coincidir com o rAthena:
```javascript
// Dentro do objeto de configuração
packetver: 20130618,

```


3. Instale as dependências:
```powershell
cd roBrowserLegacy
npm install

```



### 2. Iniciar o Proxy WebSocket

O navegador não fala TCP puro, então precisamos de um tradutor.
**Abra um NOVO terminal** e rode:

```powershell
wsproxy -p 5999

```

*(Mantenha este terminal aberto. O roBrowser vai conectar aqui, e o proxy repassa para o rAthena).*

### 3. Iniciar o Cliente Web

No terminal do projeto **roBrowserLegacy**, inicie o modo de desenvolvimento:

```powershell
npm run live

```

---

###  Resumo de Execução

Para jogar, você deve ter 4 coisas rodando simultaneamente:

1.  **Docker:** rAthena (Login/Char/Map).
2.  **RemoteClient:** `npm start` (Serve os arquivos).
3.  **WSProxy:** `wsproxy -p 5999` (Faz a ponte da conexão).
4. 🌐 **roBrowser:** `npm run live` (O site do jogo).

Acesse: `http://localhost:8080` (ou a porta indicada no terminal).
