
---

# Guia de Implantação: Ragnarok Online Web (rAthena + roBrowser)

Este guia detalha a configuração de um ambiente Ragnarok Online rodando no navegador, composto por três módulos principais:

1. **Backend:** Servidor rAthena (Docker).
2. **Asset Server:** Servidor de arquivos estáticos (Remote Client).
3. **Frontend:** Interface do cliente web (roBrowser Legacy).

> **Referência de Versão:** Este guia utiliza a versão de pacote **2013-06-18**.
> [Download do Cliente Hexed (2013-06-18)](https://github.com/Cronus-Emulator/CronusClient/tree/master/Hexeds/Ragexe/2013/2013-06-18)

---

## 1. Pré-requisitos do Sistema

Ferramentas essenciais para a execução do ambiente.

### 1.1 Node.js (Runtime JavaScript)

Necessário para executar o cliente web e o proxy de conexão.

1. Faça o download da versão **LTS** em: [nodejs.org](https://nodejs.org/en/download).
2. Valide a instalação no terminal:
```powershell
node -v
npm -v

```


3. Instale o proxy de WebSocket globalmente:
```powershell
npm install -g wsproxy

```



### 1.2 WSL 2 e Docker (Servidor)

Necessário para a virtualização do servidor rAthena.

1. **Verifique a Arquitetura do Processador:**
```powershell
if ($env:PROCESSOR_ARCHITECTURE -eq "ARM64") { Write-Host "Baixe Docker ARM" } else { Write-Host "Baixe Docker x86_64" }

```


2. **Instale o WSL 2:**
Execute o PowerShell como Administrador: `wsl --install`. Reinicie o computador após o término.
3. **Instale o Docker Desktop:**
Durante a instalação, certifique-se de marcar a opção: `Use WSL 2 instead of Hyper-V`.

---

## 2. Backend (rAthena via Docker)

Configuração do emulador para aceitar conexões via navegador.

### 2.1 Desativar Criptografia de Pacotes

O roBrowser requer a leitura de pacotes descriptografados. Edite o arquivo:
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

### 2.2 Definir Versão do Pacote

Fixar a versão do servidor para 20130618.
Edite o arquivo: `rathena\tools\docker\docker-compose.yml`

Localize a seção `environment` e ajuste a variável:

```yaml
BUILDER_CONFIGURE: "--enable-packetver=20130618"

```

### 2.3 Compilar e Iniciar

No PowerShell, navegue até a pasta do Docker (`cd rathena\tools\docker`) e execute a sequência:

```powershell
# 1. Parar containers e remover volumes órfãos
docker compose down --remove-orphans

# 2. Limpar binários antigos (garante recompilação limpa)
docker compose run --rm builder sh -c "rm -f /rathena/login-server /rathena/char-server /rathena/map-server /rathena/web-server"

# 3. Compilar o Servidor
docker compose run --rm builder

# 4. Iniciar serviços (Login, Char e Map)
docker compose up -d db login char map

```

---

## 3. Remote Client (Servidor de Arquivos)

Este projeto serve os arquivos `.grf`, áudio e dados do jogo.

### 3.1 Instalação

1. Clone ou baixe: [roBrowserLegacy-RemoteClient-JS](https://github.com/FranciscoWallison/roBrowserLegacy-RemoteClient-JS)

3.configurações crie um arquivo na raiz chamdo `.env`

exemplo:

```
PORT=3338
CLIENT_PUBLIC_URL=http://127.0.0.1:8000
NODE_ENV=development

# Cache settings
CACHE_MAX_FILES=100
CACHE_MAX_MEMORY_MB=256

```



4. Instale as dependências:
```powershell
cd roBrowserLegacy-RemoteClient-JS
npm install

```



### 3.2 Organização dos Arquivos (Assets)

Copie as pastas do seu cliente Ragnarok (baseado na versão 2013-06-18) para a raiz deste projeto. A estrutura deve ficar exatamente assim:

```text
roBrowserLegacy-RemoteClient-JS/
├── AI/            <-- Copiar do cliente original
├── BGM/           <-- Copiar do cliente original
├── System/        <-- Copiar do cliente original
├── resources/
│   ├── data.grf   <-- Copiar do cliente original
│   └── DATA.INI   <-- Criar/Editar este arquivo

```

**Conteúdo do arquivo `resources\DATA.INI`:**

```ini
[Data]
0=data.grf

```

### 3.3 Inicialização

```powershell
npm start

```

*Mantenha este terminal aberto.*

---

## 4. Frontend (Interface Web)

A aplicação cliente que roda no navegador.

### 4.1 Instalação e Configuração

1. Clone ou baixe: [roBrowserLegacy](https://github.com/MrAntares/roBrowserLegacy)
2. Instale as dependências:
```powershell
cd roBrowserLegacy
npm install

```


3. **Configurar Versão do Pacote:**
Edite o arquivo `roBrowserLegacy\applications\pwa\Config.local.js`. Se não existir, crie-o. Adicione a configuração:
```javascript
// Configuração para compatibilidade com o backend
packetver: 20130618,

```



### 4.2 Iniciar Proxy WebSocket

O navegador não se comunica diretamente via TCP, exigindo um proxy.
Abra um **novo terminal** e execute:

```powershell
wsproxy -p 5999

```

*Mantenha este terminal aberto.*

### 4.3 Iniciar Aplicação Web

No terminal do projeto **roBrowserLegacy**, inicie o servidor de desenvolvimento:

```powershell
npm run live

```

---

## Resumo de Execução

Para o funcionamento correto, quatro processos devem estar rodando simultaneamente em terminais distintos:

1. **Docker:** Backend rAthena (Login/Char/Map).
2. **RemoteClient:** `npm start` (Servidor de arquivos).
3. **WSProxy:** `wsproxy -p 5999` (Ponte de conexão).
4. **roBrowser:** `npm run live` (Interface do jogo).

Acesse o jogo através do endereço exibido no terminal do roBrowser (geralmente `http://localhost:8080`).
