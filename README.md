# Exercicio 5 - Stateful containers

Faça o seguinte teste com o container MySQL criado no exercicio anterior:

```
select count(1) from user

docker stop mysql

docker start mysql

select count(1) from user
```

O resultado do select mudou? Por que?

E se fizermos isso:

```
docker stop mysql

docker rm mysql

docker run -p 3306:3306 --name mysql -e MYSQL_ROOT_PASSWORD=rootpass -e MYSQL_USER=db_user -e MYSQL_PASSWORD=db_pass -e MYSQL_DATABASE=sample-db -d mysql:5.6.51 

select count(1) from user
```

O resultado do select mudou? Por que?

#### Stateful vs stateless

O conceito de uma aplicação stateful é que ela lembra de ao menos uma coisa desde a sua última execução, e para lembrar de algo, algum tipo de persistência é necessário, portanto uma aplicação só pode ser stateful se ela tiver algum lugar para armazenar suas informações para poder ler depois. Esse requisito faz com que aplicações stateful escalem menos que aplicações stateless.

E onde os dados são armazenados de forma persistente? Em um disco. O container tem acesso a um disco e pode persistir e ler dados dele, porém esses dados se perdem quando o container é destruido, e o que aconteceria se nós precisássemos executar vários containers da mesma aplicação acessando os mesmos dados? Como por exemplo um servidor web servindo páginas html?

Uma forma simples de tornar a aplicação stateless é delegar a persistência de seu estado para uma outra aplicação: o banco de dados. Porém isso resolve o problema da nossa aplicação e não do sistema como um todo pois o banco ainda assim vai precisar armazenar seus dados em algum lugar

#### Subindo um site simples

Vamos rodar um servidor [HTTP Apache](https://hub.docker.com/_/httpd) (antigo e muito comum na comunidade):

```
docker run --rm -d --name apache -p 80:80 httpd
```

Acesse no seu browser http://172.0.2.32/

E se quisermos customizar o html.

Crie no diretório (pode ser na home da vm), chamado **meu-blog** e junto com um arquivo **index.html** com o conteúdo que deseja. Por exemplo:

```
mkdir meu-blog && echo "<html><body><h1>Meu blog</h1></body></html>" >> meu-blog/index.html
```

Agora crie um Dockerfile dessa forma:

```
FROM httpd:2.4
COPY ./meu-blog/ /usr/local/apache2/htdocs/
```

Builde e suba sua imagem 

```
docker build --tag meu-blog .

docker run -p 80:80 --rm --name meu-blog meu-blog
```

Acesse novamente http://172.0.2.32/

E agora se quisermos alterar o conteúdo do nosso html? Como faríamos? (imagine que podem existir centenas desses containers rodando e um load balance na frente).

Apesar de servir conteúdo, essa aplicação é stateless, ela não persiste nem altera nada, somos nós que alteramos, porém cada alteração do conteúdo que ela serve requer recriar e reiniciar todos containers. Isso normalmente não chega a ser um problema, mas claro que limita a flexibilidade de alteração (imagina a cada novo texto no seu blog você ter que reiniciar todos os containers. 

#### Compartilhando diretórios entre containers

Podemos compartilhar diretórios entre o host e o container, execute novamente o servidor apache da seguinte forma (assumindo que você criou o diretório meu-blog na home):

```
docker run -p 80:80 --rm --name apache -d -v ~/meu-blog:/usr/local/apache2/htdocs/  httpd
```

Acesse novamente http://172.0.2.32/

Faça algumas modificações no html, e, sem parar o container, de um reload em http://172.0.2.32/

#### Mantendo os dados salvos do banco

Para não perdemos os dados do nosso banco, temos que salvar ele fora do container, normalmente na documentação da imagem existe a explicação de como fazer isso.

Vamos executar novamente nosso MySQL:

```
docker run -p 3306:3306 --name mysql --net=my-net -v ~/temp/mysql-data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=rootpass -e MYSQL_USER=db_user -e MYSQL_PASSWORD=db_pass -e MYSQL_DATABASE=sample-db -d mysql:5.6.51
```

Reiniciamos a aplicação (pois ela quem cria as tabelas):

```
docker restart sample-app
```

Se tiver removido o container da aplicação:

```
docker run -d -p 8080:30001 -e MYSQL_HOST=mysql --net my-net --name sample-app user/sample-app:4
```

Adicione alguns usuários em http://172.0.2.32:8080/swagger-ui.html, verique no banco que eles estão incluidos, e então remova o container do MySQL novamente, recrie-o com o mesmo volume mapeado, veja seus dados agora serão mantidos.

Como curiosidade, experimente executar um outro container MySQL apontando para o mesmo dados

```
docker run --rm -p 3307:3306 --name mysql2 --net=my-net -v ~/temp/mysql-data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=rootpass -e MYSQL_USER=db_user -e MYSQL_PASSWORD=db_pass -e MYSQL_DATABASE=sample-db -d mysql:5.6.51

docker logs -f mysql2
```

Qual o problema? Isso mostra por que é dificil escalar bancos.
