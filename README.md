# Deploy do Vaultwarden via docker compose

# Resumo  
O vaultwarden é uma implementação alternativa e leve do servidor bitwarden, compatível com a API original e escrito na linguagem Rust, perfeito para hospedar por conta própria e também é compatível com os plugins e clientes oficiais do Bitwarden.

# Justificativa  
Criamos este deploy para já trazer diversas opções pré-definidas que você provavelmente irá usar em produção. A documentação oficial do Vaultwarde traz exemplos simples e uma documentação extensa. Este repositório serve como um facilitador do deploy. Para casos de uso mais avançados, indicamos que use a documentação oficial do Vaultwarden como guia.  


# Instalação  

### Instale o docker
```
curl https://get.docker.com | sudo bash
```

### Clonar este repositório
```
git clone https://github.com/sysadminbr/vaultwarden-deploy.git
```  

### Acessar a pasta do projeto
```
cd vaultwarden-deploy
```

### Editar e ajustar os parâmetros do Vaultwarden
```
vi docker-compose.yaml

# exemplo:
version: '3'
services:
  cofre:
    image: vaultwarden/server:latest
    container_name: cofre
    volumes:
      - vaultwarden-data:/data/
    ports:
      - "8000:80"
    restart: unless-stopped
    environment:
      - DOMAIN=https://cofre.empresa.com.br
      - SENDS_ALLOWED=true
      - DISABLE_ICON_DOWNLOAD=false
      - SIGNUPS_ALLOWED=true
      - ORG_CREATION_USERS=luciano@citrait.com.br,fulano@gmail.com
      - INVITATIONS_ALLOWED=true
      - INVITATION_ORG_NAME=Minha Empresa
      - ORG_GROUPS_ENABLED=true
      - SMTP_HOST=smtp.gmail.com
      - SMTP_FROM=notificacoes@empresa.com.br
      - SMTP_FROM_NAME=Cofre de Senhas
      - SMTP_SECURITY=starttls
      - SMTP_PORT=587
      - SMTP_USERNAME=notificacoes@empresa.com.br
      - SMTP_PASSWORD=senhaSegura
      - SMTP_AUTH_MECHANISM=Plain
      - SMTP_DEBUG=false
volumes:
  vaultwarden-data:
	
```


### Realizar o deploy
```
sudo docker compose up -d
```


### (Opcional) Gerar um certificado auto-assinado para o nginx servir HTTPS
```
# altere cofre.empresa.com.br para outro domínio
openssl req -new -x509 -newkey rsa:4096 -keyout vaultwarden.key -out vaultwarden.crt -nodes -subj "/CN=cofre.empresa.com.br" -days 3650 -addext "subjectAltName=DNS:cofre.empresa.com.br"

# agora copie o certificado e chave gerados para a pasta comum de certificados
sudo cp vaultwarden.* /etc/ssl/private/

# e altere a permissão do certificado e chave para o nginx conseguir lê-los
sudo chown www-data:www-data /etc/ssl/private/vaultwarden.*
```


### Instale o nginx
```
sudo apt install nginx -y
```


### Configurar o nginx 
#### Edite o arquivo /etc/nginx/sites-enabled/default com o seguinte conteúdo
```
# onde buscar o upstream (porta do container)
upstream vaultwarden{
        server 127.0.0.1:8000;
}
# definição do endpoint http, redirecionando para https
server {
        listen 80;
        listen [::]:80;
        location / {
                rewrite ^ https://$host$uri redirect;
        }
}
# definição do endpoint https
server {
        listen 443 ssl;
        listen [::]:443 ssl ;
        ssl_certificate /etc/ssl/private/vaultwarden.crt;
        ssl_certificate_key /etc/ssl/private/vaultwarden.key;
        location / {
                proxy_set_header X-Real-IP       $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_pass http://vaultwarden;
        }
}
```

### Reinicie o serviço do nginx  
```
# testar a configuração antes de reiniciar
sudo nginx -t

# reiniciar
sudo nginx -s reload
```


### Acesse o vaultwarden pelo navegador    
  
Acesse o navegador usando o endereço https://cofre.empresa.com.br.
Lembre-se de que deve ter registrado o nome "cofre.empresa.com.br" no DNS.  


### Impedindo o auto-registro de novos usuários    
Após o deploy inicial e registro da primeira conta administrador, edite novamente o arquivo docker-compose.yaml e altere o parâmetro SIGNUPS_ALLOWED de true para false.
```
# antes
SIGNUPS_ALLOWED=false

# depois
SIGNUPS_ALLOWED=false
```
  
Agora sinalize o docker para recarregar a configuração.
```
# recarregar o container com a nova configuração
sudo docker compose up -d
```


### Download dos clientes do cofre    
Para download dos clientes do cofre (browser plugin, desktop app, mobile, ...) visite a página https://bitwarden.com/download/.

### Observações ao usar o cliente CLI  
BITWARDEN CLI - (PARA USAR COM VAULTWARDEN)  
```
Ao chamar na linha de comandos, tem que passar a env do certificado auto-assinado.

exemplo cmd  
set NODE_EXTRA_CA_CERTS=cofre.intranet.corp.crt bw login


login with apikey - Using API key environment variables

https://bitwarden.com/help/cli/
```


# Documentação Oficial  
https://github.com/dani-garcia/vaultwarden  
https://github.com/dani-garcia/vaultwarden/wiki  
https://bitwarden.com/download/  



