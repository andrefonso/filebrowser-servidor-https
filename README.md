# Configurando Acesso Externo ao File Browser com HTTPS usando Caddy e Certificado Autossinado

Este tutorial mostra como acessar o File Browser pela internet de forma segura usando HTTPS com **certificado autossinado**, ideal para quando não se quer depender do Let's Encrypt ou de domínios públicos.

---

## Requisitos

- Zorin OS (ou outro Linux)
- File Browser instalado e funcionando em `http://localhost:8080`
- Caddy instalado
- IP público fixo ou uso de DDNS (opcional)
- Redirecionamento de porta 443 no roteador para a máquina Zorin
- Permissões `sudo`

---

## 1. Gerar um certificado SSL autossinado

```bash
mkdir -p ~/certs
openssl req -x509 -newkey rsa:4096 -sha256 -days 365 -nodes \
  -keyout ~/certs/filebrowser.key \
  -out ~/certs/filebrowser.crt \
  -subj "/CN=meuarquivos.local"
```

Isso irá criar:
- `filebrowser.crt` (certificado)
- `filebrowser.key` (chave privada)

---

## 2. Instalar o Caddy (se ainda não tiver)

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

---

## 3. Configurar o Caddy com HTTPS autossinado

Edite o arquivo:

```bash
sudo nano /etc/caddy/Caddyfile
```

Insira a seguinte configuração (substitua o domínio se necessário):

```caddy
https://meuarquivos.local:443 {
    tls /home/SEU_USUARIO/certs/filebrowser.crt /home/SEU_USUARIO/certs/filebrowser.key
    reverse_proxy localhost:8080
}
```

Substitua `SEU_USUARIO` pelo seu nome de usuário do sistema.

---

## 4. Redirecionar a porta no roteador

Configure o roteador para encaminhar:

```
Porta externa: 443
IP interno: 192.168.0.142 (ou o IP do seu Zorin OS)
Porta interna: 443
Protocolo: TCP
```

---

## 5. Reiniciar o Caddy

```bash
sudo systemctl restart caddy
```

---

## 6. Acessar o File Browser

Abra o navegador e acesse:

```
https://meuarquivos.local
```

Você verá um aviso de segurança (por ser certificado autossinado). Confirme a exceção para acessar.

---

## 7. Habilitar o File Browser no boot

```bash
sudo systemctl enable filebrowser
sudo systemctl start filebrowser
```

---

## 🦆 Configurando DDNS com DuckDNS

Se você não possui um IP público fixo, pode usar o DuckDNS para registrar um domínio gratuito que sempre apontará para seu servidor, mesmo com IP dinâmico.

### 1. Criar conta no DuckDNS

- Acesse: https://www.duckdns.org
- Faça login com GitHub, Google ou outro provedor
- Registre um subdomínio (exemplo: `meuarquivos`)
- Copie seu token de autenticação

### 2. Criar diretório e script de atualização

```bash
mkdir -p ~/duckdns
cd ~/duckdns
nano duck.sh
```

Conteúdo do `duck.sh` (substitua `seusubdominio` e `seutoken`):

```bash
#!/bin/bash
echo url="https://www.duckdns.org/update?domains=seusubdominio&token=seutoken&ip=" | curl -k -o ~/duckdns/duck.log -K -
```

Torne o script executável:

```bash
chmod +x duck.sh
```

### 3. Atualizar IP automaticamente com cron

```bash
crontab -e
```

Adicione ao final do arquivo:

```bash
*/5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
```

### 4. Testar manualmente

```bash
./duck.sh
cat duck.log
```

Se funcionar, a resposta será:

```
OK
```

### 5. Usar o domínio DuckDNS

Agora você pode usar seu domínio (ex: `meuarquivos.duckdns.org`) para acessar o File Browser mesmo fora de casa.

Substitua o domínio usado no `Caddyfile` por este novo domínio.

---

## Observações

- O aviso de "conexão insegura" é esperado com certificados autossinados.
- Se quiser evitar o aviso, pode importar o certificado como confiável no navegador.
- Se você já usa DDNS, pode considerar configurar o Caddy para usar Let's Encrypt em vez de um certificado autossinado.

---

## Referências

- https://filebrowser.org
- https://caddyserver.com
- https://duckdns.org
- https://openssl.org
