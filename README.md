# Exercício 12 - Rabbit

Inicie um container do [rabbitmq](https://www.rabbitmq.com/):

```
docker run -d --rm -p 5672:5672 -p 15672:15672 --name rabbitmq -e RABBITMQ_DEFAULT_USER=user -e RABBITMQ_DEFAULT_PASS=pass rabbitmq:3-management
```

Usamos a imagem management, que sobe junto uma interface de gerenciamento do rabbit, acesse: http://172.0.2.32:15672/ e logue com usuário **user** e **senha** pass (veja que são parâmetros que passamos ao iniciar o rabbit).

O rabbit é composto por [exchanges](http://172.0.2.32:15672/#/exchanges) e [filas](http://172.0.2.32:15672/#/queues). Normalmente, várias aplicações conectam no mesmo rabbit, então temos filas e exchanges de várias aplicações. Para prover algum tipo de separação desses recursos, o rabbit tem [virtual hosts](https://www.rabbitmq.com/vhosts.html), você pode criar um virtual host no menu admin, crie um com nome **sample-vh**.

Agora crie uma exchange nesse virtual host, do tipo **fanout**. Clique nos detalhes dessa exchange e publique uma mensagem para nela, você deverá ver uma mensagem "Message published, but not routed.". Isso ocorre porque as mensagens não são armazenadas nas exchanges, você envia mensagem para elas, e elas roteiam para filas ligadas a exchange, a mensagem é armazenada na fila.

Vamos criar então uma fila nesse virtual host. Veja que no momento que você cria a fila, existem uma serie de parâmetros que pode ser informados em args, se quiser, pode adicionar alguns, por exemplo **message-ttl**. A fila sozinha não recebe nenhuma mensagem, va nos detalhes da fila que você criou e adicione um bind dessa fila com a exchange criada anteriormente. 

Volte para os detalhes da exchange que você criou, e envie novamente mensagens para ela, depois, volte aos detalhes da sua fila e veja que agora essas mensagens estão lá (se você configurou um ttl curto, pode ser que a fila já esteja vazia).

Uma fila pode fazer bind com várias exchanges e uma exchange pode ter bind de diversas filas.

Crie outras exchanges (direct, topic, headers) e outras filas, crie bind e teste o roteamento.

Desafio: temos uma fila e um requisito que, se em 30 segundos a mensagem não for consumida, temos que enviar essa mensagem para outra fila para posterior tratamento. Somente com o rabbit, como implementar esse isso? (dica, verifique os parâmetros para criação da fila).

### Rabbit na aplicação

Voltamos ao banco H2, para simplificar subir a aplicação.

Veja as alterações na aplicação, vamos usar o virtual host criado antes, e agora toda vez que criarmos um usuário, vamos enviar uma mensagem para uma exchange chamada **user_changed.exchange**, criamos duas filas, uma **gmail.user.changed** que faz bind na exchange apenas quando a routing key for gmail.com, e outra **	
log.user.changed** cuja routing key é *.

Vamos subir a aplicação na própria vm:

```
./gradlew clean build

java -jar build/libs/sample-app-0.0.12-SNAPSHOT.jar
```

A aplicação vai conectar no rabbit via localhost (quando subimos o rabbit, fizemos mapeamento da porta do container para uma porta da vm).

Acesse http://172.0.2.32:30001/swagger-ui.html e crie uns usuários (pode usar o mesmo endpoint que usamos para o teste de carga), fique vendo o log da aplicação enquanto cria usuários. Pode criar usando curl também:

```
curl localhost:30001/users/random
```

Crie alguns usuários com email do gmail e veja que tem um consumidor a mais para esse tipo de usuário.

Veja no log o consumo do evento do usuário gerado (dois consumidores).

```
2021-02-26 21:32:26.516  INFO 1229 --- [ntContainer#1-1] web.core.user.OnUsersChanged : usuario alterado, id=2f4fe6db-b259-4feb-876f-d00c11bac660, name=Joao
2021-02-26 21:33:15.751  INFO 1229 --- [ntContainer#0-4] web.core.user.OnUsersChanged : usuario com email do gmail alterado, id=73e1b95d-2e7d-4fc8-bb01-07a481bd9cf6, name=Jose
```

Enviamos a mensagem para o rabbit na criação do usuário e consumimos esse evento na mesma aplicação.

Agora crie um usuário com nome consumer_name_err (na classe OnUserChanged tem um if para que usuários com esse nome, lance uma exceção no consumidor). A aplicação vai ficar retentando consumir o usuário, mas veja no rabbit que a mensagem permanece lá. Crie um novo usuário e veja que a mensagem desse novo usuário será consumida, mesmo sendo roteada para mesma fila, por que? Quais as consequências dessa característica?



