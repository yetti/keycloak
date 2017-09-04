# Keycloak
A gem Keycloak foi desenvolvida para integrar aplicações e serviços ao sistema [Keycloak](http://www.keycloak.org/) da [Red Hat](https://www.redhat.com) para controle de usuários, autenticação, autorização e sessão.

O seu desenvolvimento foi baseado na versão 3.2 do Keycloak, cuja documentação pode ser encontrada [aqui](http://www.keycloak.org/archive/documentation-3.2.html).

## Instalação

Adicione esta linha no <b>Gemfile</b> de sua aplicação:

```ruby
gem 'keycloak'
```

Então execute:

    $ bundle

Ou instale você mesmo:

    $ gem install keycloak

## Utilização

Considerando que você já possua um ambiente do Keycloak configurado e a gem já instalada, o próximo passo é definir como será a autenticação da aplicação. O Keycloak trabalha com os principais protocolos de autenticação, tais como o OpenID Connect, Oauth 2.0 e SAML 2.0, integrando acesso a sistemas via Single-Sign On, podendo inclusive disponibilizar acessos a usuários LDAP ou Active Directory.

Ao cadastrar um Reino e também um Client no seu ambiente Keycloak, será necessário fazer o download do arquivo de instalação do Client para dentro da pasta raiz da aplicação, para que a gem obtenha as informações necessárias para interagir com o Keycloak. Para fazer esse download, basta acessar o cadastro de seu Client, clicar na aba <b>Installation</b>, selecionar <b>Keycloak OIDC JSON</b> no campo <b>Format option</b> e clicar em <b>Download</b>.

A gem possui um módulo principal chamado <b>Keycloak</b>. Dentro desse módulo há três outros módulos: <b>Client</b>, <b>Admin</b> e <b>Internal</b>.

### Module Keycloak

O módulo Keycloak possui alguns atributos e suas definições são fundamentais para o perfeito funcionamento da gem na aplicação.

```ruby
Keycloak.proxy
```

Caso o ambiente onde a sua aplicação será utilizada exija a utilização de proxy para o consumo das APIs do Keycloak, então defina-o neste atributo. Ao ser instalada, a gem cria o arquivo <b>keycloak.rb</b> em <b>config/initializers</b>. Este atributo pode ser encontrado e definido nesse arquivo.

```ruby
Keycloak.generate_request_exception
```

Este atributo serve para definir se as exceções HTTP geradas nos retornos das requisições feitas para o Keycloak serão ou não estouradas na aplicação. Caso definido como <b>false</b>, então a exceção não será estourada e a resposta HTTP será retornada para a aplicação fazer o seu próprio tratamento. O valor default deste atributo é <b>true</b>. Ao ser instalada, a gem cria o arquivo <b>keycloak.rb</b> em <b>config/initializers</b>. Este atributo pode ser encontrado e definido nesse arquivo.

```ruby
Keycloak.keycloak_controller
```

É recomendado que a sua aplicação possua um controller que centraliza as ações de sessão que o Keycloak irá gerenciar, tais como a ação de login, logout, atualização de sessão, reset de senha, entre outras. Defina neste atributo qual é o nome do controler que desempenhará esse papel. Se o seu controler chama <b>SessionController</b>, então o valor deste atributo deverá ser apenas <b>session</b>. Ao ser instalada, a gem cria o arquivo <b>keycloak.rb</b> em <b>config/initializers</b>. Este atributo pode ser encontrado e definido nesse arquivo.


```ruby
Keycloak.proc_cookie_token
```

Este atributo trata-se de um método anônimo (lâmbida). O mesmo deve ser implementado na aplicação para que a gem tenha acesso ao token de autenticação que, por sua vez, deverá ser armazenado no cookie. Ao realizar a autenticação no keycloak através da gem, o sistema deverá armazenar o token retornado no cookie do browser, como por exemplo: 
```ruby
cookies.permanent[:keycloak_token] = Keycloak::Client.get_token(params[:user_login], params[:user_password])
```
A aplicação poderá recuperar o token no cookie implementando o método <b>Keycloak.proc_cookie_token</b> da seguinte forma:
```ruby
Keycloak.proc_cookie_token = -> do
  cookies.permanent[:keycloak_token]
end
```
Desta forma, todas as vezes que a gem precisar utilizar as informações do token para consumir um serviço do Keycloak, ele irá invocar este método lâmbida.

```ruby
Keycloak.proc_external_attributes
```

O Keycloak dá a possibilidade de que novos atributos sejam mapeados no cadastro de usuários. Porém, quando esses atributos são específicos da aplicação, recomenda-se que a própria os gerencie. Para isso, a melhor solução é criar esses atributos na aplicação - exemplo: criar uma tabela no banco de dados da própria aplicação contendo as colunas representando cada um dos atributos, inserindo também nessa tabela uma coluna de identificação única (unique key), contendo na mesma o Id do usuário criado no Keycloak, indicando que esse pertencente àquele Id possui aqueles atributos.
Para que a gem tenha acesso a esses atributos, definina o atributo <b>Keycloak.proc_external_attributes</b> com um método lâmbida obtendo do <b>model</b> os atributos do usuário logado. Exemplo:
```ruby
    Keycloak.proc_external_attributes = -> do
      atributos = UsuariosAtributo.find_or_create_by(user_keycloak_id: Keycloak::Client.get_attribute('sub'))
      if atributos.status.nil?
        atributos.status = false
        atributos.save
      end
      atributos
    end
```

<b>Observações:</b> Os atributos <b>Keycloak.proc_cookie_token</b> e <b>Keycloak.proc_external_attributes</b> podem ser definidos no <b>initialize</b> do controler <b>ApplicationController</b>.


### Client

Esse módulo possui os métodos que representam as APIs de <b>endpoints</b>. Esses serviços são fundamentais para a criação e atualização de tokens, efetuação de login e logout, e, também para a obtenção de informações sintéticas de um usuário logado. O que habilita a gem a fazer uso de todos esses serviços é o arquivo de instalação do client citado anteriormente.

Vamos ao detalhamento de cada um desses métodos:

```ruby
Keycloak::Client.get_token(user, password)
```

Caso você opte por efetuar a autenticação dos usuários utilizando a tela da sua própria aplicação, então utilize esse método. Basta invocá-lo no método de login no <b>controller</b> definido com o controlador de sessão de sua aplicação, passando como parâmetro o <b>usuário</b> e a <b>senha</b> informados pelo usuário. Caso a autenticação seja válida, então será retornado um JSON contendo entre as informações principais o <b>access_token</b> e o <b>refresh_token</b>.


```ruby
Keycloak::Client.url_login_redirect(redirect_uri, response_type = 'code')
```

Para efetuar a autenticação dos usuários de sua aplicação utilizando um template configurado no Keycloak, redirecione a requisição para a url retornada nesse método. Passe como parâmetro a url que o usuário terá acesso no caso de êxito na autenticação(<b>redirect_uri</b>) e também o tipo de resposta (<b>response_type</b>), que caso não informado, a gem assumirá o valor <b>code</b>. Caso a autenticação seja bem sucedida, então será retornado um <b>code</b> que te habilitará a requisitar um token ao Keycloak.

```ruby
Keycloak::Client.get_token_by_code(code, redirect_uri)
```

Ao utilizar o método Keycloak::Client.url_login_redirect para obter um <b>code</b>, passe-o como parâmetro neste método para que o Keycloak retorne um token, efetuando assim o login do usuário na aplicação. O segundo parâmetro (redirect_uri) deve ser passado para que, ao disponibilizar um token, o Keycloak redirecione para a url informada.

```ruby
Keycloak::Client.get_token_by_refresh_token(refresh_token = '')
```

Quando o usuário já estiver logado e a sua aplicação acompanhar internamente o tempo de expiração do token fornecido pelo Keycloak, então esse método poderá ser utilizado para a renovação desse token, caso o mesmo ainda seja válido. Para isso, bastar passar como parãmetro o <b>refresh_token</b>. Caso não seja informado o <b>refresh_token</b>, a gem utilizará o <b>refresh_token</b> armazenado no cookie da aplicação.

```ruby
Keycloak::Client.get_token_introspection(token = '')
```

Esse método retorna a as informações da sessão do <b>token</b> passado como parâmetro. Entre as informações retornadas, a mais importante é o campo <b>active</b>, pois ele informa se a sessão do token passado no parâmetro é ativo ou não. Isso auxiliará a sua aplicação a controlar se a sessão do usuário logado expirou ou não. Caso nenhum token seja passado como parâmetro, a gem utilizará o último <b>access_token</b> armazenado no cookie da aplicação.

```ruby
Keycloak::Client.get_token_by_client_credentials(client_id = '', secret = '')
```

Há alguns serviços do Keycloak como <b>reset de senha</b>, <b>cadastro de usuário</b> na tela inicial da aplicação ou até mesmo autenticação seguindo o padrão <b>OAuth 2.0</b>, que a autenticação de um usuário torna-se desnecessária. Sendo assim, podemos obter um token utilizando as credenciais da sua própria aplicação (Client) cadastrada no Keycloak. Para obter esse token, deve-se passar como parâmetro desse método o <b>client_id</b> - informado pela pessoa que cadastrou sua aplicação no Keycloak - e a <b>secret</b> de sua aplicação gerado pelo Keycloak - para gerar uma <b>secret</b>, o <b>Access Type</b> do seu Client (Aplicação) deverá estar configurado como <b>confidential</b>. Caso você não passe nenhum desses parãmetros, a gem utilizará as credenciais contidas no arquivo de instalação citado anteriormente.<br>

```ruby
Keycloak::Client.logout(redirect_uri = '', refresh_token = '')
```

Quando utilizado antes da expiração da sessão do usuário logado, esse método encerra a sessão. Se o prâmetro <b>redirect_uri</b> for alimentado, então o Keycloak redirecionará a sua aplicação para a url informada após a efetuação do logout. O segundo parâmetro é o <b>refresh_token</b> obtido no momento da autenticação ou da atualização da sessão. Caso este último não seja informado, então a gem utilizará o <b>refresh_token</b> do cookie.

```ruby
Keycloak::Client.get_userinfo(access_token = '')
```

Esse método retorna informações sintéticas do usuário representado pelo <b>access_token</b> passado como parâmetro, tais como <b>sub</b> - que é o Id do usuário autenticado -, <b>preferred_username</b> - que é o nome do usuário autenticado - e <b>email</b> - que é o e-mail do usuário. Cado o parâmetro <b>access_token</b> não seja informado, então a gem obterá essa informação no cookie.

```ruby
Keycloak::Client.url_user_account
```

Retorna a <b>url</b> para acesso ao cadastro de usuários do Reino do arquivo de instalação (keycloak.json). Para ter acesso a tela, o Keycloak exigirá a autenticação do usuário. Após logado, e caso tenha permissão, o usuário terá acesso a suas informações cadastrais podendo inclusive alterá-las.

```ruby
Keycloak::Client.has_role?(user_role, access_token = '')
```

O método <b>has_role?</b> decodifica o JWT <b>access_token</b> e verifica se o usuário dono do token possui o <b>role</b> informado no parâmetro <b>user_role</b>. Caso o <b>access_token</b> não seja informado, então a gem utilizará o <b>access_token</b> do cookie.

```ruby
Keycloak::Client.user_signed_in?(access_token = '')
```

Esse método verifica se o <b>access_token</b> passado no parâmetro ainda está ativo. Para verificar se o usuário está ativo ou não, internamente a gem invoca o método <b>get_token_introspection</b>. Caso o <b>access_token</b> não seja informado, então a gem utilizará o <b>access_token</b> do cookie.

```ruby
Keycloak::Client.get_attribute(attribute_name, access_token = '')
```

Esse método decodifica o JWT <b>access_token</b> e retorna o valor do atributo de nome passado no parâmetro <b>attribute_name</b>. Esse atributo pode ser um <b>mapper</b> - cadastrado na seção <b>Mappers</b> do cadastro do <b>Client</b> do Reino. Caso o <b>access_token</b> não seja informado, então a gem utilizará o <b>access_token</b> do cookie.

```ruby
Keycloak::Client.token
```

Retorna o último token autenticado armazenado no cookie. Quando na aplicação é implementado o método <b>Keycloak.proc_cookie_token</b> e um usuário faz a autenticação da aplicação, esse método retornará o token desse usuário.

```ruby
Keycloak::Client.external_attributes
```

Quando implementado o método <bKeycloak.proc_external_attributes</b>, o método <b>external_attributes</b> o retornará. A finalidade desse método é retornar os atributos específicos da aplicação não mapeados no Keycloak.
