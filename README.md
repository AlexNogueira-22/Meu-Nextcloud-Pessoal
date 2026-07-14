# Meu-Nextcloud-Pessoal
### Autor: Alex Nogueira
Depois de muito pesquisar e de varias tentativas, após consegui fazer meu Drive Local em Nuvem com exito, fiz esse breve tutorial para criar um servidor Nextcloud pessoal com Docker e Tailscale no Windows/WSL com a experiencia que adquiri. 

### 🚀 Introdução

Construi esse guia para documentar a jornada completa para construir um **servidor Nextcloud pessoal e seguro** diretamente no seu computador com Windows, utilizando o WSL2, Docker e Tailscale. O objetivo principal é ter um "Google Drive" próprio, totalmente gratuito, utilizando o seu armazenamento pessoal de forma ilimitada e acesso privado de qualquer lugar, sem precisar abrir portas no roteador .

### 🎯 O que você vai construir

1.  **Nextcloud**: A plataforma de código aberto para armazenamento em nuvem, sincronização de arquivos e colaboração .
2.  **Docker & Docker Compose**: Ferramentas para criar e gerenciar os containers do Nextcloud e do banco de dados MariaDB de forma simples e eficiente .
3.  **Tailscale**: A "cola mágica" que cria uma VPN privada (mesh VPN) entre seus dispositivos. Com ela, você acessa seu Nextcloud de qualquer lugar com total segurança, sem precisar de domínio público ou configurações complexas de roteador .

---

### 📋 Pré-requisitos

Antes de começar, certifique-se de ter:

*   **Sistema**: Windows 10 ou 11 (64-bit) com suporte a virtualização ativado.
*   **Hardware**: Mínimo de 8GB de RAM e, pelo menos, 4 núcleos de CPU .
*   **Contas**: Uma conta Google, Microsoft ou GitHub para criar seu usuário no Tailscale.

---

### 🛠️ PASSO A PASSO DA INSTALAÇÃO

#### 1. Preparando o Ambiente no Windows: WSL2 e Docker

O Nextcloud vai rodar dentro de um ambiente Linux no seu Windows (WSL2) .

1.  **Ativar o WSL**:
    *   Abra o **PowerShell como Administrador** e execute:
        ```powershell
        dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
        wsl --install -d Ubuntu
        ```
    *   Reinicie o computador quando solicitado .

2.  **Instalar o Docker Desktop**:
    *   Baixe e instale o **Docker Desktop** para Windows. Durante a instalação, certifique-se de selecionar a opção para usar o **WSL 2**, pois isso é fundamental para o desempenho e integração .
    *   Após a instalação, abra o Docker Desktop, vá em **Configurações > Recursos > Integração WSL** e ative a integração com a sua distribuição Ubuntu.
    *   **Importante**: Verifique se o Docker Desktop está configurado para iniciar automaticamente com o Windows .

#### 2. Instalando e Configurando o Tailscale

O Tailscale será a nossa ponte segura para acessar o Nextcloud de qualquer lugar, sem expor portas do seu computador à internet pública.

1.  **Instalar no Ubuntu (WSL)**:
    *   Abra o terminal do Ubuntu e execute o comando de instalação:
        ```bash
        curl -fsSL https://tailscale.com/install.sh | sh
        ```
        

2.  **Autenticar e Conectar**:
    *   Ainda no terminal Ubuntu, inicie o Tailscale:
        ```bash
        sudo tailscale up
        ```
    *   Siga o link gerado no navegador para fazer login com sua conta Google, Microsoft ou GitHub. Isso vinculará seu computador à sua rede privada do Tailscale (Tailnet) .
    *   Verifique se está conectado: `tailscale ip` deve mostrar um IP (ex: `100.64.0.1`).

3.  **Configurar no Windows e Celular**:
    *   Baixe e instale o Tailscale também no seu PC Windows e no seu celular (Android/iOS), fazendo login com a **mesma conta**. Eles agora fazem parte da sua VPN privada.

#### 3. Criando e Configurando o Nextcloud

Este é o coração do projeto, onde instalaremos o Nextcloud usando o Docker Compose.

1.  **Criar a Estrutura de Pastas**:
    *   No terminal Ubuntu, crie um diretório para o projeto e navegue até ele:
        ```bash
        mkdir ~/nextcloud-server
        cd ~/nextcloud-server
        ```

2.  **Criar o Arquivo `docker-compose.yml`**:
    *   Este arquivo define os serviços (containers) que vão rodar. Usamos um modelo simples e funcional, com o banco de dados MariaDB .
    *   Execute o comando abaixo para criar o arquivo:
        ```bash
        cat > docker-compose.yml << 'EOF'
        services:
          db:
            image: mariadb:10.11
            restart: always
            command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW
            volumes:
              - ~/mariadb-data:/var/lib/mysql
            environment:
              - MYSQL_ROOT_PASSWORD=senha_root_segura
              - MYSQL_PASSWORD=senha_db_segura
              - MYSQL_DATABASE=nextcloud
              - MYSQL_USER=nextcloud
        
          app:
            image: nextcloud:latest
            restart: always
            ports:
              - "8080:80"
            depends_on:
              - db
            volumes:
              - ~/nextcloud-data:/var/www/html
            environment:
              - MYSQL_PASSWORD=senha_db_segura
              - MYSQL_DATABASE=nextcloud
              - MYSQL_USER=nextcloud
              - MYSQL_HOST=db
        EOF
        ```

    *   **⚠️ Atenção**: Substitua `senha_root_segura` e `senha_db_segura` por senhas fortes .

3.  **Iniciar o Nextcloud**:
    *   No terminal Ubuntu, dentro da pasta `~/nextcloud-server`, execute:
        ```bash
        docker compose up -d
        ```
    *   Este comando baixa as imagens e inicia os containers em segundo plano .

4.  **Verificar se está rodando**:
    ```bash
    docker compose ps
    ```
    Ambos os serviços (`app` e `db`) devem estar com o status **"Up"**.

5.  **Acessar e Configurar pela Primeira Vez**:
    *   No seu navegador, abra `http://localhost:8080`.
    *   Você verá a tela de configuração do Nextcloud. Crie uma conta de administrador (usuário e senha) e, se solicitado, as configurações de banco de dados já estarão preenchidas com os dados do `docker-compose.yml`. Clique em **"Instalar"** .
    *   **Dica**: Se quiser instalar a partir do zero, pare e remova os containers com `docker compose down -v` e repita o processo .

#### 4. Configurando o Nextcloud para Funcionar com o Tailscale

Por padrão, o Nextcloud só aceita conexões vindas de `localhost`. Precisamos adicionar o IP do Tailscale à lista de domínios confiáveis .

1.  **Descobrir o IP do Tailscale**:
    *   No terminal Ubuntu, execute:
        ```bash
        tailscale ip
        ```
    *   Anote o IP (ex: `100.65.2.0`).

2.  **Adicionar o IP aos Domínios Confiáveis**:
    *   O arquivo de configuração `config.php` fica dentro do container. Vamos copiá-lo para editar:
        ```bash
        docker cp nextcloud-server-app-1:/var/www/html/config/config.php ~/nextcloud-server/config.php
        ```
    *   Edite o arquivo com `nano ~/nextcloud-server/config.php` e localize a seção `'trusted_domains'`. Adicione o IP do Tailscale com a porta `:8080`:
        ```php
        'trusted_domains' =>
          array (
            0 => 'localhost:8080',
            1 => '100.65.4.39:8080',  // <-- ADICIONE ESTA LINHA
          ),
        ```
    *   Salve (`Ctrl+X`, `Y`, `Enter`) e copie o arquivo de volta para o container:
        ```bash
        docker cp ~/nextcloud-server/config.php nextcloud-server-app-1:/var/www/html/config/config.php
        ```
    *   Reinicie o container para aplicar a mudança:
        ```bash
        docker compose restart app
        ```

3.  **Testar o Acesso Remoto**:
    *   No seu **celular** (conectado ao Tailscale), abra o navegador e acesse: `http://100.65.4.39:8080`.
    *   A página de login do Nextcloud deve aparecer! 🎉 Isso significa que você pode acessar seus arquivos de qualquer lugar do mundo, desde que o Tailscale esteja ativo no servidor e no seu dispositivo.

#### 5. (Opcional) Sincronizando com o App Desktop no Windows

1.  Baixe e instale o cliente Nextcloud para Windows.
2.  Ao configurar, use o IP do Tailscale (`http://100.65.4.39:8080`) como endereço do servidor .
3.  Selecione uma pasta de sincronização no seu HD secundário (ex: `D:\Nextcloud`).
4.  Recomendo marcar a opção **"Use arquivos virtuais"** para economizar espaço em disco.

#### 6. Mantendo o Servidor Rodando

O servidor precisa estar ligado para ser acessado. Para garantir que ele continue rodando mesmo com o terminal do Ubuntu fechado:

1.  **Solução Rápida**: Abra o Docker Desktop, vá para a aba "Containers" e inicie seus containers se eles pararem.
2.  **Solução Permanente**: No **PowerShell do Windows (como Admin)**, execute:
    ```powershell
    wsl --exec dbus-launch true
    ```
    Isso impede que o WSL seja desligado automaticamente. Para uma solução definitiva, crie/edite o arquivo `C:\Users\SeuUsuario\.wslconfig` com o conteúdo:
    ```ini
    [wsl2]
    vmIdleTimeout=-1
    ```

### 🧠 Resolução de Problemas (Troubleshooting)

Durante a jornada, alguns erros podem aparecer. Aqui estão os mais comuns e como resolvê-los.

*   **Erro: `port is already allocated` (Porta 8080 em uso)**
    *   **O que é**: Outro programa no Windows está usando a porta 8080.
    *   **Como resolver**: No **PowerShell**, execute `netstat -ano | findstr :8080` para descobrir o PID do processo. Depois, mate-o com `taskkill /PID [NUMERO_DO_PID] /F`. Em seguida, tente `docker compose up -d` novamente.

*   **Erro: `ERR_CONNECTION_REFUSED` (Conexão recusada)**
    *   **O que é**: O navegador não conseguiu se conectar ao servidor.
    *   **Como resolver**:
        *   Verifique se os containers estão rodando: `docker compose ps`.
        *   Verifique os logs para ver se há erros internos: `docker compose logs app`.
        *   Se você está no Windows e o `localhost` não funciona, tente usar o IP do WSL, descoberto com `ip addr show eth0` no terminal Ubuntu (ex: `http://172.31.208.253:8080`).

*   **Erro: `503 Service Unavailable` e/ou `Configuration was not read correctly`**
    *   **O que é**: Problema com o arquivo `config.php`.
    *   **Como resolver**: Isso pode ocorrer se o arquivo estiver corrompido. Siga os passos da seção 4 para recriá-lo . Às vezes, parar tudo com `docker compose down -v` e recomeçar do zero é a solução mais rápida.

### 🏁 Conclusão

Com esse pequeno tutorial você é capaz de construiu seu próprio servidor de nuvem seguro e acessível de qualquer lugar, sem custos com hospedagem ou serviços terceiros. Este projeto demonstra o poder do software livre (Nextcloud), do Docker e de ferramentas modernas de rede como o Tailscale para criar soluções pessoais robustas.

### 📚 Referências e Agradecimentos

Este guia foi inspirado e construído a partir de diversas fontes e discussões da comunidade:

*   Projetos de exemplo no GitHub.
*   Tutoriais sobre Nextcloud com Docker e Tailscale.
*   Documentação oficial do Nextcloud AIO para Windows.
*   Agradecimento especial à comunidade que compartilha conhecimento e facilita a vida de quem está começando .
