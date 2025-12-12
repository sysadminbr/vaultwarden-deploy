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
      - DOMAIN=https://cofre.empresa.corp
      - SENDS_ALLOWED=true
      - DISABLE_ICON_DOWNLOAD=false
      - SIGNUPS_ALLOWED=true
      - ORG_CREATION_USERS=luciano@citrait.corp,fulano@gmail.com
      - INVITATIONS_ALLOWED=true
      - INVITATION_ORG_NAME=Minha Empresa
      - ORG_GROUPS_ENABLED=true
      - SMTP_HOST=smtp.gmail.com
      - SMTP_FROM=notificacoes@empresa.corp
      - SMTP_FROM_NAME=Cofre de Senhas
      - SMTP_SECURITY=starttls
      - SMTP_PORT=587
      - SMTP_USERNAME=notificacoes@empresa.corp
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
# gerar a chave privada da autoridade
openssl genrsa -out ca.key 4096

# criar a autoridade
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
  -out ca.pem -subj "/CN=CA-COFRE" -addext "basicConstraints = critical, CA:TRUE"

# gerando o certificado assinado pela CA
openssl req -new -x509 -subj "/CN=cofre.citrait.corp" -CA ca.pem -CAkey ca.key \
	-addext "subjectAltName=DNS:cofre.citrait.corp" -days 365 \
	-newkey 2048 -keyout cofre.key -out cofre.pem -nodes

# concatena a CA com o certificado de servidor para produzir uma cadeia (fullchain)
cat cofre.pem ca.pem > fullchain.pem

# move os certificados para a pasta comum
sudo mv {fullchain.pem,cofre.key} /etc/ssl/private/

# ajusta permissão dos certificados
sudo chown -R www-data:www-data /etc/ssl/private/{fullchain.pem,cofre.key}
```


### Instale o nginx
```
sudo apt install nginx -y
```


### Configurar o nginx 
#### Execute o bloco de código abaixo para criar uma configuração padrão do nginx para o vaultwarden
```
cat << "EOF" > default
# onde buscar o upstream (porta do container)
upstream vaultwarden{
        server 127.0.0.1:8000;
        keepalive 2;
        zone vaultwarden-default 64k;
}

map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      "";
}

# definição do endpoint http, redirecionando para https
server {
        listen 80;
        listen [::]:80;
        server_name _;
        location / {
                rewrite ^ https://$host$uri redirect;
        }
}
# definição do endpoint https
server {
        listen 443 ssl http2;
        listen [::]:443 ssl ;
        server_name _;
        ssl_certificate /etc/ssl/private/fullchain.pem;
        ssl_certificate_key /etc/ssl/private/cofre.key;  
        location / {
	        client_max_body_size 500M;
	        proxy_http_version 1.1; 
	        proxy_set_header Upgrade $http_upgrade;  
	        proxy_set_header Connection $connection_upgrade;  
	        proxy_set_header Host $host;  
	        proxy_set_header X-Real-IP $remote_addr;
	        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	        proxy_set_header X-Forwarded-Proto $scheme;
					proxy_pass http://vaultwarden;   
        }
}
EOF
sudo mv default  /etc/nginx/sites-enabled/default
```

### Reinicie o serviço do nginx  
```
# testar a configuração antes de reiniciar
sudo nginx -t

# reiniciar
sudo nginx -s reload
```


### Acesse o vaultwarden pelo navegador    
  
Acesse o navegador usando o endereço https://cofre.empresa.corp.
Lembre-se de que deve ter registrado o nome "cofre.empresa.corp" no DNS.  


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
Ao chamar na linha de comandos, é possível usar de forma interativa ou automática.  
Independendente da forma que usar, precisará baixar a CA e apontá-la para ser confiável. infelizmente o nodejs não lê as configurações de certificado do windows.
Vamos supor que você baixou o certificado da autoridade CA (arquivo CA-COFRE.crt) e pôs na mesma pasta do bw.

Exemplo no powershell do windows  
$env:NODE_EXTRA_CA_CERTS="CA-COFRE.crt"
$env:BW_CLIENTID="user.1df9ab67-5069-4ca4-b782-1cf34a25783e"
$env:BW_CLIENTSECRET="MumllzNjB65ckyQPgpJqbj7k9tDRQk"
$env:BW_MASTER="senhasegura"

# Aponte a URL do cofre
.\bw.exe config server https://cofre.citrait.corp

# Autentique com sua chave API
.\bw login --apikey

# Desbloqueie o Cofre e salva a saída (chave de sessão) na variável
$env:BW_SESSION=(.\bw.exe unlock --passwordenv BW_MASTER --raw)

# Agora o cofre está logado e desbloqueado para executar os comandos que precisa.
.\bw list items --search administrator
.\bw get item 99ee88d2-6046-4ea7-92c2-acac464b1412
bw get password "Domain Admin" --raw

# Criando uma senha
# para obter a lista de campos json disponíveis, execute 'bw get template item'
$tio = ConvertTo-Json ([PSCustomObject]@{"type"=1; "name"="tiolulu creds"; "notes"="Some notes about this item."; "login"=@{"username"="tiolulu";"password"="123"}})
$enc = $tio | .\bw encode
.\bw create item $enc

# Buscando e alterando uma senha
$creds = .\bw get item tiolulu | ConvertFrom-Json
$creds.login.password = 'passuord'
$creds | ConvertTo-Json | .\bw encode | .\bw edit item $creds.id

# Para executar um servidor de api rest pelo bw
# https://bitwarden.com/help/vault-management-api/
.\bw serve

# Consome a api para listar os itens
$items = Invoke-RestMethod -uri http://localhost:8087/list/object/items?search=tiolulu

# Edita a senha e salva no cofre
$tiolulu = $items.data.data | where {$_.login.username -eq "tiolulu"}
$item_id = $tiolulu.id
$tiolulu.login.password = "pazzword"
Invoke-RestMethod -Method PUT -uri http://localhost:8087/object/item/$item_id -ContentType 'application/json' -body (convertTo-Json $tiolulu)
```


# Documentação Oficial  
https://github.com/dani-garcia/vaultwarden  
https://github.com/dani-garcia/vaultwarden/wiki  
https://bitwarden.com/download/  
https://bitwarden.com/help/cli/



