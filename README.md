# Gitlab + AD FS + SAML

Foi solicitada uma ferramenta para controle de versionamento. Optamos por utilizar o GitLab por ser open-source, gratuíto e com possibilidade de ser hospedado internamente. Devido as políticas de segurança e a o dinamismo do gerenciamento de usuários, se fez necessária integração com o AD, e isso foi feito utilizando SAML. Desta forma cumprem-se os padrões solicitados na LGPD e políticas internas da xxxxxxx.

## Instalação

É necessário instalar o curl, perl, ca-certificates e adicionar o repositório do GitLab.

```bash
sudo apt-get update
sudo apt-get install -y curl ca-certificates perl debian-archive-keyring gnupg apt-transport-https

sudo curl -fsSL https://packages.gitlab.com/gitlab/gitlab-ce/gpgkey | gpg --dearmor > /usr/share/keyrings/gitlab_gitlab-ce-archive-keyring.gpg

echo deb https://packages.gitlab.com/gitlab/gitlab-ce/debian bullseye main > /etc/apt/source.list.d/gitlab_gitlab-ce.list

apt-get update
apt-get install gitlab-ce
```

## Configuração
Para a conclusão do projeto, os seguintes requisitos precisam ser atendidos:

#1 Autenticação com AD;

```python
vim /etc/gitlab/gitlab.rb

#Procure, descomente e altere a próxima linha para o habilitar o autenticação via SAML
gitlab_rails['omniauth_allow_single_sign_on'] = ['saml']

#Procure, descomente e altere a próxima linha para desabilitar a criação de usuários na tela de login
gitlab_rails['omniauth_block_auto_created_users'] = false

#Procure, descomente e altere as próximas linhas para o habilitar o link das informações dos usuários
gitlab_rails['omniauth_auto_link_ldap_user'] = true
gitlab_rails['omniauth_auto_link_saml_user'] = true


#Adicione as seguintes linhas e insira os parâmetro adequados:

gitlab_rails['omniauth_providers'] = [
    {
      name: "saml", # apenas para identificar o privder
      label: "GitLab xxxxxxx", # texto do botão de login via SAML
      args: {
        assertion_consumer_service_url: "https://gitlab.xxxxxxx.com.br/users/auth/saml/callback", # url do site + /users/auth/saml/callback
        idp_cert_fingerprint: "aa5a9c3cdfc8b99372da21efe32b09c585f9dae5" # fingerprint do Ceriticado de AUtenticação de token do AD FS
        idp_sso_target_url: "https://adfs.xxxxxx.com.br/adfs/ls", # url de login do AD FS
        name_identifier_format: "urn:oasis:names:tc:SAML:1.1:nameid-format:emailAdress", # atributo que será enviado pelo AD FS para identificar de forma única cada conta
        attribute_statements: {      # mapeamento dos valores que serão enviados pelo AD FS
               email: ['http://schemas.xmlsoap.org/ws/2005/05/identity/claims/mail'],
               first_name: ['http://schemas.xmlsoap.org/ws/2005/identity/claims/givenname'],
               last_name: ['http://schemas.xmlsoap.org/ws/2005/identity/claims/surname'],
               name: ['http://schemas.xmlsoap.org/ws/2005/identity/claims/displayname'],
               username: ['http://schemas.xmlsoap.org/ws/2005/identity/claims/name'],
```

#2 HTTPS com certificado wildcard da xxxxxx;

```python
vim /etc/gitlab/gitlab.rb

#Procure, descomente e altere a próxima linha para o URL configurado no DNS
external_url 'https://gitlab.xxxxxx.com.br' 

#Procure, descomente e altere a próxima linha para que o tráfego HTTP seja redirecionado para HTTPS
nginx['redirect_http_to_https'] = true 

#Procure, descomente e altere a próxima linha para desabilitar a integração com o Let's Encrypt, pois, vamos utilizar um certificado já existente
letsencrypt['enable'] = false 
```

#3 Apenas usuários dos grupos TI e DO devem ter acesso;                                                     
No gerenciador do AD FS precisa ser configurada a Confiaça da Terceira Parte Confiável, bem como as Claims que serão enviadas.

-> Adicionar Confiança da Terceira Parte Confiável...                                   
-> Com reconhecimento de declaração                           
-> Inserir dados sobre a terceira parte confiável manualmente            
-> Habilitar suporte para o protocolo WebSSO do SAML 2.0               
-> Identificador da confiança da terceira parte confiável = https://gitlab.xxxxxxx.com.br          
-> Permitir um grupo específico e adicionar TI e DO;


#4 O nome dos usuários deve corresponder ao "name" do AD;

-> Editar Política de Emissão de Declaração                
-> Precisa ser criada uma regra para converter um ingoing claim de e-mail para outgoing claim com emailaddress
```python

c:[Type == "http://schemas.microsoft.com/ws/2008/06/identity/claims/windowsaccountname", Issuer == "AD AUTHORITY"]
 => issue(store = "Active Directory", types = ("http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress"), query = ";mail;{0}", param = c.Value);

```
-> É necessário criar uma regra para converter um outgoing claim com emailaddress para NameID

```python

c:[Type == "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress"]
 => issue(Type = "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier", Issuer = c.Issuer, OriginalIssuer = c.OriginalIssuer, Value = c.Value, ValueType = c.ValueType, Properties["http://schemas.xmlsoap.org/ws/2005/05/identity/claimproperties/format"] = "urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress");

```
-> Por fim, uma regra com os atributos que devem ser enviados

```python

c:[Type == "http://schemas.microsoft.com/ws/2008/06/identity/claims/windowsaccountname", Issuer == "AD AUTHORITY"]
 => issue(store = "Active Directory", types = ("http://schemas.xmlsoap.org/ws/2005/05/identity/claims/mail", "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname", "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname", "http://schemas.microsoft.com/identity/claims/displayname", "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name"), query = ";mail,givenname,surname,displayname,name;{0}", param = c.Value);

```

#5 O logout deve finalizar a sessão de autenticação via SAML;

Nas propriedades da Confiança da Terceira Parte Confiável, em Pontos de Extremidade (Endpoints) precisa ser adicionado um SAML para Logoff, Ligação = POST, URL https://adfs.xxxxxxx.com.br/adfs/ls/?wa=wsignout1.0

#6 Não deve ser possível realizar login de outras formas, além do SAML;                           
 Após logar com uma conta de administrador, em settings, general, sign-up restrictions, deve ser desmarcada opção sign-up enabled.                 
-> Em sign-in restrictions deve ser desmarcada opção "Allow password authentication for the web interface", marcada opção "Enabled OAuth authentication sources" GitLab xxxxxx



## Autor
[Juarez Prates](https://gitlab.xxxxxx.com.br/JuarezPrates)
