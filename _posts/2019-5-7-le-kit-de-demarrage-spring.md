---
layout: post
title: Spring Boot, Travis, Heroku, CodeCov. Merci !
comments: true
---

Nous allons voir dans ce post comment créer une API REST très rapidement avec Spring. Si ce n'était que ça, il me faudrait juste trois lignes, du coup on va y ajouter l'intégration continue avec Travis et le déploiement continue vers Heroku. Le tout avec une petite couche de métrique sous CodeCov.

La première étape consiste à initialiser son projet Spring via le super [Initializr](https://start.spring.io/) que Spring nous met à disposition.

![Spring Initializr]({{ site.baseurl }}/images/seed-spring/initializr-spring.png)

En dépendance on ajoute uniquement "Web" afin de créer un service HTTP GET "Hello World" (désolé pour l'originalité). Pour le gestionnaire de dépendances, j'utilise Maven pour ce tutorial mais vous pouvez très bien effectuer la même chose sous Gradle.
Une fois votre nom de projet défini, vous pouvez générer le projet et dézipper la structure dans le répertoire de votre choix.

Vous pouvez désormais ouvrir le projet dans votre IDE préféré, télécharger les dépendances avec la commande `mvn install` et lancer le jar packagé sous le répertoire `target/` avec la commande `java -jar votrejar-0.0.1-SNAPSHOT.jar`

On va rapidement créer un controller avec une méthode GET "Hello World" et écrire un test d'intégration qui appelle notre méthode.

Voici notre controller :

```java
@RestController
public class HelloController {

    @GetMapping
    public String hello() {
        return "Hello World!";
    }
}
```

Ainsi que son test d'intégration associé

```java
import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.core.IsEqual.equalTo;

@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment =  SpringBootTest.WebEnvironment.RANDOM_PORT)
public class HelloControllerTests {

    @LocalServerPort
    private int port;

    private URL baseUrl;

    @Autowired
    private TestRestTemplate template;

    @Before
    public void setUp() throws MalformedURLException {
        this.baseUrl = new URL("http://localhost:" + port + "/");
    }

    @Test
    public void whenCallGet_ThenReturnsHelloWorld() {

        ResponseEntity<String> response = template.getForEntity(baseUrl.toString(), String.class);
        assertThat(response.getBody(), equalTo("Hello World!"));
    }
}
```

Etape numéro 2, nous allons brancher l'intégration continue Travis afin de nous assurer que le code que l'on commit compile et que les tests passent :

- Lancer la commande `mvn -N io.takari:maven:0.7.6:wrapper` à la racine de votre projet afin de récupérer le maven wrapper pour que Travis puisse l'utiliser lors de sa compilation
- Aller sur [Travis](https://travis-ci.org/) & on se connecte avec GitHub
- En haut à droite sous Settings, vous devriez voir la liste de vos projets GitHub, activer le toggle sur votre projet

  ![Activer TravisCI]({{ site.baseurl }}/images/seed-spring/activer-travis-ci.png)

- A la racine de votre projet, créer un fichier `.travis.yml` avec le contenu suivant :

```yaml
language: java
jdk: openjdk8
```

Maintenant on le commit & on ouvre l'onglet _"Dashboard"_ sous l'interface web de Travis. Il faut attendre une petite minute le temps que Travis se réveille, un live terminal devrait apparaître. Le build devrait être passé, le bonhomme Travis est vert ! Regardons un peu ce que Travis a lancé pour nous.

1. `./mvnw install -DskipTests=true -Dmaven.javadoc.skip=true -B -V` &rarr; Il télécharge les dépendances & compile notre projet
2. `./mvnw test -B` &rarr; Il lance notre suite de tests

Tout ça sans rien lui avoir stipulé mis à part que nous utilisons Java 8. Well done Travis! :smiley:

Maintenant c'est bien beau d'avoir un projet qui compile, mais on en veut un peu plus. On veut un projet qui est déployé ! [Heroku](https://www.heroku.com/) nous voilà :sunglasses:

Première étape, on se créé un compte. Maintenant vous pouvez vous connecter et créer une nouvelle application. Utiliser la méthode de déploiement via GitHub et activer le déploiement automatique en ayant bien coché la case "Attendre que le déploiement continue soit correctement passé" (Bah oui! on va déployer seulement si Travis n'a pas planté :smiley:). Vous devriez avoir un déploiement qui ressemble à :

![Deploiement heroku]({{ site.baseurl }}/images/seed-spring/deploiement-heroku.png)

Vous pouvez maintenant faire un commit afin de s'assurer que ce commit est correctement récupéré par Heroku & déployé. De même que pour Travis, il y a un petit délai pour que Heroku se lance.
Via la page Heroku vous pouvez cliquer sur "Open app" afin d'obtenir votre URL de déploiement. Pour cette exemple, notre service est bien accessible ici: [https://spring-boot-seed.herokuapp.com/](https://spring-boot-seed.herokuapp.com/). Magique non ? :clap:

Dernière étape, on va lancer une analyse de la couverture de notre suite de tests à chaque build de Travis afin de vérifier que nous avons pas trop dérivé !

Pour cela, on va utiliser [Codecov](https://codecov.io/). Vous pouvez vous y connecter avec GitHub et créer le projet en le selectionnant dans la liste.

Ensuite, dans notre projet, ajouter le plugin JaCoCo à votre _pom.xml_ dans la section `<build><plugins>` :

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.2</version>
  <executions>
    <execution>
	  <goals>
	    <goal>prepare-agent</goal>
	  </goals>
	</execution>
	<execution>
	  <id>report</id>
	  <phase>test</phase>
	  <goals>
	    <goal>report</goal>
	  </goals>
	</execution>
  </executions>
</plugin>
```

Puis pour finir on ajoute l'appel à Codecov au sein du build Travis en ajoutant :

```yaml
after_success:
  - bash <(curl -s https://codecov.io/bash)
```

Dans votre fichier .travis.yml

On commit le tout et une fois le build Travis passé, la couverture de tests est disponible avec une chouette interface &rarr; [https://codecov.io/gh/wadinj/spring-boot-seed](https://codecov.io/gh/wadinj/spring-boot-seed)

Vous pouvez retrouver le projet initialisé sous GitHub : [https://github.com/wadinj/spring-boot-seed](https://github.com/wadinj/spring-boot-seed).

N'hésitez pas à le forker et avoir un flot de développement aux petits oignons ! :tada:

Si vous avez des soucis, n'hésitez pas à commenter, je sortirais ma :wrench: !
