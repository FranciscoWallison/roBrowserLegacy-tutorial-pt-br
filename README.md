# Ragnarok Online no navegador — rAthena + roBrowser Legacy

Guia para subir o ambiente completo e entrar no jogo pelo navegador. São três peças:

| Peça | O que faz | Onde roda |
| --- | --- | --- |
| **rAthena** | O servidor do jogo (login, char, map) e o banco | Docker |
| **RemoteClient-JS** | Serve os arquivos do cliente (GRF, música, tabelas) e faz a ponte WebSocket → TCP para o rAthena | Node, porta 3338 |
| **roBrowser Legacy** | A interface do jogo | Node (Vite), porta 3000 |

Versão de pacote usada em todo o guia: **20130618**. É a que o roBrowser fala e a que o cliente
[Hexed 2013-06-18](https://github.com/Cronus-Emulator/CronusClient/tree/master/Hexeds/Ragexe/2013/2013-06-18)
usa. Com versões diferentes o login falha sem mensagem clara.

> Os comandos estão em PowerShell (Windows). No Linux e no macOS é o mesmo, sem a parte do WSL.

---

## Início rápido

Se você já tem Docker, Node e um cliente Ragnarok 2013-06-18 na máquina, são quatro passos. Se não tem,
veja [Instalando os pré-requisitos](#instalando-os-pré-requisitos) antes.

### 1. rAthena no Docker

```powershell
git clone https://github.com/rathena/rathena
cd rathena\tools\docker
```

Crie `docker-compose.override.yml` nessa pasta, para compilar na versão de pacote certa sem editar o
arquivo do rAthena:

```yaml
services:
  builder:
    environment:
      BUILDER_CONFIGURE: "--enable-packetver=20130618"
```

Compile e suba. A compilação é a parte demorada do guia — deixe rodando:

```powershell
docker compose run --rm builder
docker compose up -d db login char map
```

Está pronto quando `docker compose logs map | Select-String "Map Server is now online"` responder.

> O `builder` só compila quando falta algum binário. Se depois disso você mudar o packetver, apague os
> binários antes de compilar de novo, senão nada muda:
> `Remove-Item ..\..\login-server, ..\..\char-server, ..\..\map-server, ..\..\web-server -ErrorAction SilentlyContinue`

### 2. Servidor de assets (RemoteClient-JS)

```powershell
git clone https://github.com/FranciscoWallison/roBrowserLegacy-RemoteClient-JS
cd roBrowserLegacy-RemoteClient-JS
npm install
```

Crie o arquivo `.env` na raiz do projeto com o mínimo para este guia (o repositório traz um
`.env.example` com todas as opções, inclusive cache e ajustes de produção):

```ini
PORT=3338
NODE_ENV=development

# A interface roda na 3000 (Vite); sem esta origem na lista de CORS o navegador
# bloqueia todo download de asset. Use localhost, não 127.0.0.1 — o CORS compara a origem exata.
CLIENT_PUBLIC_URL=http://localhost:3000

# Ponte WebSocket → TCP para o rAthena. Sem isso não existe login pelo navegador.
ENABLE_WSPROXY=true

# Arquivos soltos do cliente, que não estão dentro do GRF. Aponte para a pasta do SEU cliente.
DATA_OVERRIDE_PATH=C:/RagnarokOnline/data
BGM_PATH=C:/RagnarokOnline/BGM
SYSTEM_PATH=C:/RagnarokOnline/System
AI_PATH=C:/RagnarokOnline/AI
```

Crie `resources/DATA.INI` apontando para o seu GRF. O caminho pode ser **absoluto**, então o arquivo de
3 GB não precisa ser copiado para dentro do projeto:

```ini
[Data]
0=C:\RagnarokOnline\data.grf
```

Suba:

```powershell
npm start
```

Pronto quando aparecer `Server ready on http://localhost:3338 | WS Proxy: /ws/`. Antes disso ele imprime
um relatório de validação — se algum caminho do `.env` estiver errado, ele diz qual.

### 3. Interface (roBrowser Legacy)

```powershell
git clone https://github.com/MrAntares/roBrowserLegacy
cd roBrowserLegacy
npm install
```

Crie `applications/pwa/Config.local.js`:

```javascript
window.ROConfigLocal = {
	// Sem isto o jogo baixa os assets da infraestrutura pública do roBrowser, e não do seu servidor.
	remoteClient: 'http://127.0.0.1:3338/',
	skipIntro: true,
	servers: [
		{
			display: 'Meu servidor',
			address: '127.0.0.1',
			port: 6900,
			version: 25,
			langtype: 12,
			packetver: 20130618,
			renewal: true,
			// O rAthena ofusca os IDs de pacote do map-server. Com false, login e char funcionam e o
			// mapa derruba a sessão no primeiro pacote.
			packetKeys: true,
			// A ponte WebSocket é o próprio servidor de assets, na 3338.
			socketProxy: 'ws://127.0.0.1:3338/ws/',
			adminList: [2000000]
		}
	]
};
```

Suba a interface:

```powershell
npm run pwa
```

### 4. Criar uma conta e entrar

```powershell
docker exec rathena-db mariadb -uragnarok -pragnarok ragnarok -e "INSERT INTO login (userid, user_pass, sex, email, group_id) VALUES ('teste', 'teste123', 'M', 'a@localhost', 0);"
```

Abra **http://localhost:3000/applications/pwa/index.html**, entre com `teste` / `teste123`, escolha um PIN
de 4 dígitos, crie o personagem e jogue.

> As senhas ficam em texto puro no banco (`use_MD5_passwords: no`, padrão do rAthena). Serve para testar na
> sua máquina; não reutilize essas credenciais em nada exposto na internet.

---

## Conferir se está tudo de pé

```powershell
# servidor de assets: status, validação e quantos arquivos foram indexados
curl.exe http://127.0.0.1:3338/api/health

# portas do rAthena (login 6900, char 6121, map 5121)
Test-NetConnection 127.0.0.1 -Port 6900

# containers de pé
cd rathena\tools\docker; docker compose ps
```

## Parar

```powershell
# rAthena (mantém o banco)
cd rathena\tools\docker; docker compose down

# apaga o banco: contas e personagens somem
docker compose down -v
```

O servidor de assets e a interface param com `Ctrl+C` nos terminais deles.

---

## Por que cada ajuste existe

Sem qualquer um destes, algo quebra — quase sempre em silêncio.

| Ajuste | Onde | Por quê |
| --- | --- | --- |
| `--enable-packetver=20130618` | `docker-compose.override.yml` | O compose do rAthena compila com 20211103. Versões diferentes mudam o layout dos pacotes e o login falha. O override deixa o arquivo versionado intocado. |
| `packetKeys: true` | `Config.local.js` | O rAthena ofusca os IDs de pacote do map-server (`PACKET_OBFUSCATION`, ligado por padrão). Com `false`, o mapa derruba a sessão no primeiro pacote. Alternativa antiga: desligar a ofuscação no servidor e recompilar — não é mais preciso. |
| `remoteClient` e `socketProxy` | `Config.local.js` | Os padrões apontam para `grf.robrowser.com` e `connect.robrowser.com`, que são públicos. Sem trocar, você testa a infraestrutura dos outros. |
| `CLIENT_PUBLIC_URL` | `.env` | A origem da interface precisa estar na lista de CORS, senão o navegador bloqueia todo download de asset. |
| `ENABLE_WSPROXY=true` | `.env` | Liga a ponte WebSocket → TCP. O navegador não fala TCP. |
| `DATA_OVERRIDE_PATH` | `.env` | `msgstringtable.txt` e as tabelas traduzidas não estão no GRF. Sem isso a interface mostra `NO MSG 1900` e botões em coreano. |
| `BGM_PATH`, `SYSTEM_PATH`, `AI_PATH` | `.env` | Música, fontes e AI também não estão no GRF. São servidos direto da pasta do cliente, só leitura. |

Nada disso precisa ser copiado para dentro dos projetos: o GRF entra por caminho absoluto no `DATA.INI` e
as pastas do cliente por variável de ambiente.

---

## Solução de problemas

| Sintoma | Causa | O que fazer |
| --- | --- | --- |
| "Disconnected from Server" ao entrar no mapa, carregamento em 100% | Ofuscação de pacotes | `packetKeys: true` no `Config.local.js` |
| "O servidor ainda reconhece seu último log-in" | A sessão anterior caiu sem deslogar | Esperar cerca de um minuto |
| Nenhum asset carrega, erro de CORS no console | A origem da interface não está na lista | `CLIENT_PUBLIC_URL=http://localhost:3000` no `.env` |
| `NO MSG 1900`, botões em coreano | Tabelas traduzidas ausentes | `DATA_OVERRIDE_PATH` apontando para a pasta `data` do seu cliente |
| Login falha sem mensagem clara | Packetver diferente entre cliente e servidor | Os dois têm que ser `20130618` |
| Servidor de assets não sobe e reclama de um caminho | Pasta citada no `.env` não existe | Corrigir o caminho ou remover a variável |
| Item aparece como "Unknown Item" | As tabelas do cliente de 2013 não conhecem itens novos que o rAthena entrega | Cosmético. Dá para completar as tabelas com o `item_db` do rAthena |
| Editei um arquivo do cliente e nada mudou no navegador | O servidor guarda o arquivo em cache e o roBrowser salva no armazenamento do navegador o que baixa | Reiniciar o servidor de assets e limpar os dados do site no navegador |

---

## Instalando os pré-requisitos

### Node.js

Baixe a versão **LTS** em [nodejs.org](https://nodejs.org/en/download) — o servidor de assets exige
**Node 22.12 ou mais novo**. Confira:

```powershell
node -v
npm -v
```

### WSL 2 e Docker Desktop (Windows)

```powershell
# PowerShell como Administrador
wsl --install
```

Reinicie, instale o [Docker Desktop](https://docs.docker.com/desktop/setup/install/windows-install/)
mantendo marcada a opção `Use WSL 2 instead of Hyper-V`, reinicie de novo e confira:

```powershell
wsl -l -v
docker --version
```

### Cliente Ragnarok 2013-06-18

Você precisa de um cliente instalado para ter o `data.grf` e as pastas `data/`, `BGM/`, `System/` e `AI/`.
O guia não distribui esses arquivos.

---

## Perguntas comuns

**Preciso mesmo de três terminais?** Dois ficam abertos (assets e interface); o Docker roda em segundo
plano. O `wsproxy` separado que guias antigos pedem não é mais necessário: a ponte WebSocket vem embutida
no servidor de assets.

**Dá para servir a interface pelo próprio servidor de assets?** Ele tem essa opção
(`ENABLE_STATIC_SERVE=true` com `ROBROWSER_PATH`), e ela serve as páginas, mas o login ainda não funciona
assim: o roBrowser importa o pacote `rijndael-js`, que só existe em CommonJS, e o navegador não carrega
isso sem um empacotador. Por isso a interface sobe pelo Vite (`npm run pwa`).

**Posso usar outro GRF?** Sim, desde que seja versão 0x200 ou 0x300 sem criptografia DES. Liste quantos
quiser no `DATA.INI`, um por linha — o menor número vence quando dois arquivos têm o mesmo caminho.

---

## Referências

- [roBrowserLegacy-RemoteClient-JS](https://github.com/FranciscoWallison/roBrowserLegacy-RemoteClient-JS) — servidor de assets, com a documentação completa das variáveis de ambiente
- [roBrowser Legacy](https://github.com/MrAntares/roBrowserLegacy) — cliente web
- [rAthena](https://github.com/rathena/rathena) — emulador
