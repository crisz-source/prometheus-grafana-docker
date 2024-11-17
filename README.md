# Prometheus & Grafana com Docker
Este projeto foi feito pela equipe da alura e modifiquei para realizar um monitoramento da aplicação com grafana e prometheus.

Para rodar o projeto no ubuntu 22.04, será necessário das seguintes tecnologias instaladas:
- Docker
- Docker Compose
- Java => Openjdk-8-jdk
- Suba os containers da aplicação com o comando, todas as configurações já foram feitas para rodar perfeitamente. Informei as configurações de ambiente do que eu fiz
```bash
docker-compose up -d
```

# Configuração do ambiente

- Entre no diretório app e faça o build da aplicação feita em java
```bash
sudo apt install maven -y
mvn clean package
```

- Aguarde o build, se ocorrer tudo certo irá receber a seguinte mensagem: 
```bash
[INFO] Replacing main artifact with repackaged archive
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  41.006 s
[INFO] Finished at: 2024-11-16T23:08:04-03:00
[INFO] ------------------------------------------------------------------------
``` 

- Feito isso, o build gerou um arquivo .jar e execute o seguiinte comando para aplicação subir com sucesso:
```bash
#!/bin/bash
java -jar -Xms128M -Xmx128M -XX:PermSize=64m -XX:MaxPermSize=128m -Dspring.profiles.active=prod target/forum.jar

# saída esperada:
2024-11-16 23:09:48.971  INFO 55941 --- [         task-1] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
2024-11-16 23:09:49.046  INFO 55941 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2024-11-16 23:09:49.047  INFO 55941 --- [           main] d.s.w.p.DocumentationPluginsBootstrapper : Context refreshed
2024-11-16 23:09:49.089  INFO 55941 --- [           main] d.s.w.p.DocumentationPluginsBootstrapper : Found 1 custom documentation plugin(s)
2024-11-16 23:09:49.140  INFO 55941 --- [           main] s.d.s.w.s.ApiListingReferenceScanner     : Scanning for api listing references
2024-11-16 23:09:49.425  INFO 55941 --- [           main] DeferredRepositoryInitializationListener : Triggering deferred initialization of Spring Data repositories…
2024-11-16 23:09:49.883  INFO 55941 --- [           main] DeferredRepositoryInitializationListener : Spring Data repositories initialized!
2024-11-16 23:09:49.893  INFO 55941 --- [           main] br.com.alura.forum.ForumApplication      : Started ForumApplication in 7.475 seconds (JVM running for 7.983)

# Entre na url da aplicação-api
http://localhost:8080/topicos
```

# Habilitando o actuator para expor as métricas da jvm

Entre neste link: https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html#actuator.enabling para poder instalar a seguinte depedência:
```bash

	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-actuator</artifactId>
	</dependency>

```

- Abra o arquivo pom.xml e cole a depedência.
- Abra o diretório /src/main/resources
- Abra o arquivo application-prod.propeties e adicione:
```bash
# actuator
management.endpoint.health.show-details=always # vai mostrar todos os detalhes
#management.endpoints.web.exposure.include=* # todas as informações relacionadas a jvm e o gerenciamento da aplicação, descomente caso você queira
management.endpoints.web.exposure.include=health,info,metrics,prometheus # informando os endpoint que queira ver com o prometheus
```
- Vai ser necessário rebuildar a aplicação dentro do diretório app/ e executar o script do banco.
```bash
mvn clean package
java -jar -Xms128M -Xmx128M -XX:PermSize=64m -XX:MaxPermSize=128m -Dspring.profiles.active=prod target/forum.jar
```

- Após rebuildar a aplicação e popular o banco, entre na url: http://localhost:8080/actuator 
```bash
# Saída esperada do actuator
 {
  "_links": {
    "self": {
      "href": "http://localhost:8080/actuator",
      "templated": false
    },
    "health-path": {
      "href": "http://localhost:8080/actuator/health/{*path}",
      "templated": true
    },
    "health": {
      "href": "http://localhost:8080/actuator/health",
      "templated": false
    },
    "info": {
      "href": "http://localhost:8080/actuator/info",
      "templated": false
    },
    "metrics-requiredMetricName": {
      "href": "http://localhost:8080/actuator/metrics/{requiredMetricName}",
      "templated": true
    },
    "metrics": {
      "href": "http://localhost:8080/actuator/metrics",
      "templated": false
    }
  }
}
```

- Em cada url, conseguirá ver as métricas

# Expondo as métricas para o Prometheus
- Aqui, vou utilizar o **micrometer.io** para enviar e que consiga ver as métricas mais legíveis para o prometheus
- Adicionei a seguinte dependencia na apalicação java para instalar o micrometer no arquivo pom.xml
```bash

<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId>
  <version>${micrometer.version}</version>
</dependency>

``` 

- Adicionei uma configuração adicional do prometheus abaixo de actuator
```bash
#actuator
management.endpoint.health.show-details=always # vai mostrar todos os detalhes
management.endpoints.web.exposure.include=health,info,metrics,prometheus # informando os endpoint que queira ver com o prometheus


#prometheus
management.metrics.enable.jvm=true # informar as métricas da jvm
management.metrics.export.prometheus.enabled=true # habilitando a exportação das metricas para o prometheus
management.metrics.distribution.sla.http.server.requests=50ms, 100ms, 200ms, 300ms, 500ms, 1s
management.metrics.tags.application=app-forum-api # configuração de uma tag, quando expor as métricas, as métricas vai ter a seguinte tag (app-forum-api)

``` 
- Será necessário realizar o rebuild novamente, e aplicar o script para subir a aplicação
- Aṕos rebuildar e popular o banco, entre na url do prometheus: http://localhost:8080/actuator/prometheus

# Métricas personalizadas
- Criei 2 métricas de autenticação de usuário, uma para sucesso e outra para erro.
- Da seguinte forma, abra a classe de autenticação de usuário no seguinte diretório: **app/src/main/java/br/com/alura/forum/controller/AutenticacaoController.java**
- Esta classe, basicamente quando um usuário passa um login e senha correta a classe retorna um toke, caso ao contrário, vai retorar um erro.

- Criei alguns atributos, e será necessário de algumas dependencias no código.
- Será necessário de um método para fazer essas autenticações. A metrica utilizada é uma métrica Counter, e neste caso adicionei ao try cat esse incremento
-  Código final
```bash
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;

@RestController
@RequestMapping("/auth")
@Profile(value = {"prod", "test"})
public class AutenticacaoController {
	
	Counter authUserSuccess;
	Counter authUserErrors;

	public AutenticacaoController(MeterRegistry registry) {
    authUserSuccess = Counter.builder("auth_user_success") // endpoint => auth_user_success
        .description("usuarios autenticados")
        .register(registry);
        
    authUserErrors = Counter.builder("auth_user_error") // endpoint => auth_user_error
        .description("erros de login")
        .register(registry);
}

	@Autowired
	private AuthenticationManager authManager;
	
	@Autowired
	private TokenService tokenService;
	
	@PostMapping
	public ResponseEntity<TokenDto> autenticar(@RequestBody @Valid LoginForm form) {
		UsernamePasswordAuthenticationToken dadosLogin = form.converter();
		
		try {
			Authentication authentication = authManager.authenticate(dadosLogin);
			String token = tokenService.gerarToken(authentication); 	
			authUserSuccess.increment();	
			return ResponseEntity.ok(new TokenDto(token, "Bearer"));
			
		} catch (AuthenticationException e) {
			authUserErrors.increment();
			return ResponseEntity.badRequest().build();
		}

		
	}
}
``` 

- Depois de criado essas métricas personalizadas, será necessário realizar um build novamente da aplicação
- Volte para url: http://localhost:8080/actuator/prometheus e pesquise por: **user**
- Note que as métricas de autenticação foi criada com sucesso!
- Faça uma requisição com o postman nessa seguinte url:  localhost:8080/auth 
- Em Authorization no postman, utilize o type como **bearer token**
- Header, no campo **KEY**, coloque a opção **Content-Type** e em **VALUE** adicione **application/json**
- Body, adicione o seguinte json:
```bash
{
    "email":"teste.batatinha123@gmail.com
    "senha":"q1w2e3r4"
}
``` 
- Clique em **Send** para disparar a requisição, e postman vai retornar o token de authenticação do tipo Bearer
- Agora com usuário authenticado, note que nas métricas ao atualizar o navegador, na métrica **auth_user_success_total{application="app-forum-api",} 1.0** informa que um usuário teve sua conta authenticada
- Em caso de teste, pode ficar apertando infinitamente **Send** para verificar a captura da metrica.
- Note que agora, no meu teste teve 15 usuários authenticado: **auth_user_success_total{application="app-forum-api",} 15.0**
- Testando o erro, adicione um caracter a mais na sua senha e atualize o navegador das metricas
- Note que a métrica de erro foi capturada: **auth_user_error_total{application="app-forum-api",} 5.0** ou seja, 5 usuários tiverem problemas com a sua conta

# Conteinerizando a aplicação
- Até este momento a aplicação estava rodando na máquina local, apenas o redis e mysql estava rodando em containers.
- Fiz algumas alterações para a apalicação conseguir fazer a conexão com o container redis e o container do mysql
- no diretório: **app/src/main/resources/application-prod.properties** troquei onde estava localhost do redis e mysql, para o nome do serviço do docker-compose
```bash
# Redis cache config 
spring.cache.type=redis
spring.redis.host=redis-forum-api
spring.redis.port=6379

# datasource
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://mysql-forum-api:3306/forum
spring.datasource.username=forum
spring.datasource.password=Bk55yc1u0eiqga6e
``` 
- Será necessário refazer o build para empacotar a aplicação em container.

# Subindo o docker-compose da aplicação e prometheus
- As configurações necessárias para subir aplicação com o prometheus em docker, está no arquivo docker-compose. Basta apenas subir a aplicação
```bash
docker-compose up -d
```

- Caso receba um erro do proxy, informando que a porta 80 esta sendo usada, siga estes passos para que funcione corretamente:
```bash
# Verifique qual processo a porta esta sendo utilizada
sudo lsof -i :80

# para encerrar o process:
sudo kill -9 <PID> 

# Execeute novmente o comando docker-compose up -d, caso o erro ainda persiste. É um sinal que o apache2 esta rodando no sistema
# Desabilite temporiamente o apache2, assim que der um boot na máquina, ele vai ser reiniciado novamente na porta 80
sudo systemctl stop apache2

# verifque se a porta ainda esta sendo usada
sudo lsof -i :80 # se nao retornar nada, a porta 80 está liberada. 


# OBS: Caso queira desabilitar o apache2 mesmo dando boot, execute este comando:
sudo systemctl disable apache2

# Iniciando o apache2 novamente
sudo systemctl enable apache2

# Verificando se esta ativo
sudo systemctl status apache2
``` 
- entre na seguinte url da aplicação: http://localhost/topicos
- não foi necessário informar a porta por conta do proxy que está rodando do ngnix, que esta fazendo o redirecionamento
- acessando as metricas: http://localhost/metrics
- Acessando o prometheus: http://localhost:9090


# Métricas com o prometheus
em contrução...