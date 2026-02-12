
# Instalação: WSL 2 + Docker Desktop (Windows 11)

Este documento cobre a preparação do ambiente (WSL 2) e a instalação do container engine (Docker Desktop), garantindo a melhor performance e compatibilidade para desenvolvimento.

---

### **Passo 1: Verificação de Arquitetura**

Antes de baixar o instalador, é crucial confirmar a arquitetura do processador para evitar erros de compatibilidade.

1. Abra o **PowerShell** e execute o seguinte script de validação:
```powershell
if ($env:PROCESSOR_ARCHITECTURE -eq "ARM64") { 
    Write-Host "✅ Baixe: Docker Desktop for Windows - Arm" -ForegroundColor Green 
} else { 
    Write-Host "✅ Baixe: Docker Desktop for Windows - x86_64" -ForegroundColor Green 
}

```


2. **Anote o resultado:**
* Se **x86_64**: Use o instalador padrão (Intel/AMD).
* Se **Arm**: Use a versão específica para Arm (Snapdragon/Apple Silicon via Parallels).



---

### **Passo 2: Instalação do Motor (WSL 2)**

O Docker no Windows depende do *Windows Subsystem for Linux (WSL)* para rodar containers Linux nativamente sem a sobrecarga de uma VM tradicional.

1. Abra o **PowerShell como Administrador**.
2. Execute o comando de instalação automática:
```powershell
wsl --install

```


3. **Aguarde:** O Windows baixará o Kernel Linux e a distribuição padrão (Ubuntu).
4. **⚠️ REINICIALIZAÇÃO OBRIGATÓRIA:** * Assim que o comando finalizar, **reinicie o computador imediatamente**.
* *Nota:* Ao ligar, uma janela preta (terminal) pode abrir pedindo para criar um `UNIX username` e `password`. Crie um usuário simples (ex: `dev`).



---

### **Passo 3: Instalação do Docker Desktop**

Com o motor (WSL) pronto, instale a interface de gerenciamento.

1. **Download:**
* Acesse a documentação oficial: [Docker Desktop for Windows](https://docs.docker.com/desktop/setup/install/windows-install/)
* Clique no botão correspondente ao resultado do **Passo 1** (geralmente *Docker Desktop for Windows - x86_64*).


2. **Instalação:**
* Execute o instalador `.exe`.
* Na tela de configuração, **mantenha marcada** a opção:
> ☑️ **Use WSL 2 instead of Hyper-V (Recommended)**


* Siga até finalizar e feche o instalador.



---

### **Passo 4: Validação Final**

Confirme se o Docker está rodando e integrado ao WSL.

1. Abra o **Docker Desktop** e aguarde o ícone na barra de tarefas (perto do relógio) parar de piscar ou ficar verde.
2. No PowerShell, rode o comando para verificar a integração:
```powershell
wsl --list --verbose

```


### Start Projeto
No seu terminal (PowerShell), entre na pasta do Docker primeiro:

```powershell
cd rathena\tools\docker

```

Rode tudo assim:

```powershell
# 1. Parar tudo (remove containers e redes órfãs)
docker compose down --remove-orphans

# 2. Limpar binários antigos (Usando o próprio contexto do compose)
docker compose run --rm builder sh -c "rm -f /rathena/login-server /rathena/char-server /rathena/map-server /rathena/web-server"

# 3. Recompilar
docker compose run --rm builder

# 4. Iniciar servidores (em background)
docker compose up -d db login char map

```



