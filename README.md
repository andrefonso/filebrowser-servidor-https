# Configurando Acesso Externo com HTTPS ao File Browser no Zorin OS

Este tutorial descreve como tornar o File Browser acessível pela internet com conexão segura via HTTPS, utilizando Nginx como proxy reverso e Certbot (Let's Encrypt) para geração automática de certificado SSL.

> ⚠️ O File Browser já deve estar instalado e funcionando em rede local, servindo arquivos a partir da pasta:
>
> `/home/andre/Compartilhado/`

---

## Requisitos

- Zorin OS com o File Browser já instalado.
- Nome de domínio válido (ex: `meuservidor.dyndns.org`) apontando para seu IP público.
- Portas 80 e 443 redirecionadas no seu roteador para o IP local do Zorin OS.
- Acesso root ou sudo.

---

## 1. Instalar o Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

---

## 2. Criar configuração Nginx para proxy reverso

Crie um arquivo de configuração do site:

```bash
sudo nano /etc/nginx/sites-available/filebrowser
```

Conteúdo (substitua `meuservidor.dyndns.org` pelo seu domínio):

```nginx
server {
    listen 80;
    server_name meuservidor.dyndns.org;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Ative a configuração com um link simbólico:

```bash
sudo ln -s /etc/nginx/sites-available/filebrowser /etc/nginx/sites-enabled/
```

Teste a configuração do Nginx:

```bash
sudo nginx -t
```

Reinicie o Nginx:

```bash
sudo systemctl restart nginx
```

---

## 3. Instalar o Certbot e configurar HTTPS com Let's Encrypt

```bash
sudo apt install certbot python3-certbot-nginx -y
```

Obtenha e configure o certificado SSL:

```bash
sudo certbot --nginx -d meuservidor.dyndns.org
```

Durante o processo:
- Aceite os termos de uso.
- Escolha a opção de redirecionamento automático de HTTP para HTTPS.

Teste a renovação automática:

```bash
sudo certbot renew --dry-run
```

---

## 4. (Opcional) Ativar firewall UFW e liberar acesso

```bash
sudo apt install ufw -y
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

---

## 5. Acesso externo ao File Browser

Agora o File Browser estará acessível de qualquer lugar via HTTPS:

```
https://meuservidor.dyndns.org
```

---

## 6. Segurança adicional recomendada

- Altere as credenciais padrão no painel do File Browser.
- Use senhas fortes.
- Verifique periodicamente os certificados SSL com:

```bash
sudo certbot certificates
```

---

## Referências

- https://filebrowser.org
- https://certbot.eff.org
- https://nginx.org
