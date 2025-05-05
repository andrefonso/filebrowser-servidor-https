# I) Acesso Seguro ao File Browser via HTTPS (Interno e Externo) com DuckDNS

# Configuração do File Browser com HTTPS autossinado, acessível pela rede local e remotamente pela internet usando DDNS (DuckDNS) em um servidor Linux.

Este tutorial ensina como configurar acesso remoto seguro ao **File Browser** instalado em um servidor Linux (Zorin OS, por exemplo), usando **HTTPS com certificado autossinado** e **DDNS com DuckDNS**.



---

## ✅ Requisitos

- Servidor com **Zorin OS** (ou similar) com o **File Browser** já instalado.
- Pasta a ser compartilhada: `/home/andre/Compartilhado/`
- Domínio DDNS já criado no DuckDNS: `filecontrol.duckdns.org`
- Acesso ao roteador para redirecionar portas.
- Conexão com a internet.

---

## 🌐 1. Configurar DDNS com DuckDNS

### Criar estrutura de diretório e script de atualização

```bash
mkdir -p ~/duckdns
nano ~/duckdns/duck.sh
```

### Conteúdo do `duck.sh`:

```bash
#!/bin/bash
echo url="https://www.duckdns.org/update?domains=filecontrol&token=SEU_TOKEN_DUCKDNS&ip=" | curl -k -o ~/duckdns/duck.log -K -
```

Substitua `SEU_TOKEN_DUCKDNS` pelo seu **token pessoal** disponível no site do [DuckDNS](https://www.duckdns.org).

### Tornar o script executável

```bash
chmod +x ~/duckdns/duck.sh
```

### Automatizar a execução com cron

```bash
crontab -e
```

Adicione a linha abaixo ao final do arquivo:

```cron
*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
```

Isso fará com que o IP seja atualizado a cada 5 minutos.

---

## 🔁 2. Redirecionar a porta no roteador

Acesse o painel do seu roteador e redirecione a porta 443 para o IP local do servidor (ex: `192.168.0.142`) e a porta do File Browser (ex: 8080).

**Exemplo de redirecionamento de porta:**
- Porta externa: `443`
- IP interno: `192.168.0.142`
- Porta interna: `8080`
- Protocolo: `TCP`

---

## 🔒 3. Gerar Certificado HTTPS Autossinado

### Criar arquivo de configuração

```bash
nano ~/duckdns/filebrowser-cert.conf
```

### Conteúdo do arquivo:

```ini
[req]
default_bits       = 2048
prompt             = no
default_md         = sha256
req_extensions     = req_ext
distinguished_name = dn

[dn]
CN = filecontrol.duckdns.org

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1   = filecontrol.duckdns.org
DNS.2   = filecontrol.local
IP.1    = 192.168.0.142
```

### Gerar o certificado:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048   -keyout /etc/ssl/private/filebrowser.key   -out /etc/ssl/certs/filebrowser.crt   -config ~/duckdns/filebrowser-cert.conf
```

---

## ⚙️ 4. Configurar o File Browser para usar HTTPS

Edite o serviço do File Browser para usar o certificado:

```bash
sudo nano /etc/systemd/system/filebrowser.service
```

**Exemplo de configuração:**

```ini
[Unit]
Description=File Browser
After=network.target

[Service]
User=andre
ExecStart=/usr/local/bin/filebrowser -r /home/andre/Compartilhado/   --address 0.0.0.0 --port 8080   --cert /etc/ssl/certs/filebrowser.crt   --key /etc/ssl/private/filebrowser.key
Restart=always

[Install]
WantedBy=multi-user.target
```

> Certifique-se de que o caminho do binário `filebrowser` esteja correto.

### Recarregar e reiniciar o serviço:

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable filebrowser
sudo systemctl restart filebrowser
```

---

## 🔍 5. Acessar o File Browser

### Acesso pela rede local:

```
https://192.168.0.142:8080
```

> Será necessário aceitar o certificado autossinado no navegador.

### Acesso remoto pela internet:

```
https://filecontrol.duckdns.org
```

> Você também verá um aviso de segurança ao usar um certificado autossinado. Basta clicar em "Avançado" > "Prosseguir assim mesmo".

---

## 🧪 Testes e validações

- Acesse da sua rede local e de fora da sua casa.
- Verifique se a página de login do File Browser aparece.
- Faça login com suas credenciais configuradas.

---

## ✅ Finalizado

Agora você pode acessar seu servidor de arquivos com segurança, tanto **em rede local** quanto **pela internet**, usando o File Browser com HTTPS.

# II) Acesso Seguro ao File Browser via HTTPS com DuckDNS e Caddy (Let's Encrypt)

O tutorial a seguir mostra como configurar o File Browser em um servidor Linux (Zorin OS ou similar) para ser acessado via HTTPS tanto pela rede local quanto externamente pela internet, utilizando um certificado **válido** da Let's Encrypt com auxílio do **Caddy Server** e **DuckDNS** como serviço DDNS.

---

## ✅ Requisitos

- Servidor Linux com o File Browser já instalado
- Pasta de compartilhamento: `/home/andre/Compartilhado/`
- Nome de domínio no [DuckDNS](https://www.duckdns.org/): `filecontrol.duckdns.org`
- Token DuckDNS
- IP reservado no roteador para o servidor (ex: `192.168.0.142`)
- Redirecionamento de porta 443 → 8080 no roteador
- Conexão com a internet

---

## 🌐 1. Configurar DuckDNS

### 1.1 Criar domínio

Acesse [https://www.duckdns.org](https://www.duckdns.org), crie uma conta e registre o domínio `filecontrol`.

Copie seu **token** e anote.

### 1.2 Atualizador automático do IP

Crie diretório para scripts do DuckDNS:

```bash
mkdir -p ~/duckdns
nano ~/duckdns/duck.sh
```

Conteúdo do arquivo `duck.sh`:

```bash
#!/bin/bash
echo url="https://www.duckdns.org/update?domains=filecontrol&token=SEU_TOKEN&ip=" | curl -k -o ~/duckdns/duck.log -K -
```

Substitua `SEU_TOKEN` pelo seu token real do DuckDNS.

Dê permissão de execução:

```bash
chmod +x ~/duckdns/duck.sh
```

Agende no `crontab` para rodar a cada 5 minutos:

```bash
crontab -e
```

Adicione a linha ao final:

```bash
*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
```

---

## 🔁 2. Redirecionar Porta no Roteador

Acesse seu roteador e redirecione:

- Porta externa: `443`
- IP interno: `192.168.0.142` (ou IP do seu servidor)
- Porta interna: `8080`
- Protocolo: `TCP`

---

## 📦 3. Instalar o Caddy Server

Execute:

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

---

## ⚙️ 4. Configurar o Caddy como Proxy Reverso

Edite o arquivo de configuração:

```bash
sudo nano /etc/caddy/Caddyfile
```

Conteúdo:

```
filecontrol.duckdns.org {
    reverse_proxy localhost:8080
}
```

Salve e feche.

Reinicie o Caddy:

```bash
sudo systemctl restart caddy
```

O Caddy irá automaticamente:

- Obter um certificado HTTPS da Let's Encrypt
- Redirecionar requisições para o File Browser

---

## 🔒 5. Configurar o File Browser

Inicie o File Browser apontando para a pasta compartilhada:

```bash
filebrowser -r /home/andre/Compartilhado/ -p 8080
```

Ou configure o serviço via systemd:

```bash
sudo systemctl enable filebrowser
sudo systemctl start filebrowser
```

---

## 🌍 6. Acesso

- **Acesso externo:** https://filecontrol.duckdns.org
- **Acesso local:** https://192.168.0.142

Ambos funcionarão com certificado HTTPS válido.

---

## 📌 Observações

- Se usar o Caddy, **não precisa configurar certificados manualmente**.
- O navegador **não exibirá alertas de segurança** pois o certificado da Let's Encrypt é reconhecido.
- Se quiser expor múltiplos serviços, o Caddy também gerencia múltiplos domínios facilmente.

---

## ✅ Concluído

Agora você pode acessar seu servidor com o File Browser de forma segura, localmente ou de qualquer lugar pela internet!
