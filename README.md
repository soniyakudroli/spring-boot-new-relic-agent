Source : http://www.kubrynski.com/2014/12/include-java-agent-in-standalone-spring.html
May not work : https://github.com/spring-projects/spring-boot/issues/6626

Including Java agent in standalone Spring Boot application 
December 31, 2014
Recently at DevSKiller.com we've decided to move majority of our stuff to simple containers. It was pretty easy due to use of Spring Boot uber-jars, but the problem was in NewRelic agents which should have to be included separately. That caused uncomfortable situation so we decided to solve it by including NewRelic agent into our uber-jar applications. If you also want to simplify your life please follow provided instructions :)

At first we have to add proper dependency into our pom.xml descriptor:
<dependency>
  <groupiId>com.newrelic.agent.java<<groupId>
  <artifactId>newrelic-agent</artifactId>
  <version>3.12.1</version>
  <scope>provided</scope>
</dependency>

Now since we have proper jar included into our project it's time to unpack the dependency to have all necessary classes in our application jar file:
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-dependency-plugin</artifactId>
  <version>2.9</version>
  <executions>
    <execution>
      <phase>prepare-package</phase>
      <goals>
         <goal>unpack-dependencies</goal>
      </goals>
      <configuration>
        <includeArtifactIds>newrelic-agent</includeArtifactIds>
        <outputDirectory>${project.build.outputDirectory}</outputDirectory>
      </configuration>
    </execution>
  </executions>
</plugin>

After this step we've all agent related classes accessible directly from our jar. But still the file cannot be used as an agent jar. There are some important manifest entries that have to be present in every agent jar. The most important is the Premain-Class attribute specifying main agent class including premain() method. In case of NewRelic it's also important to include Can-Redefine-Classes and Can-Retransform-Classes attributes. The easiest way to do that is to extend maven-jar-plugin configuration:

<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-jar-plugin</artifactId>
  <version>2.5</version>
  <configuration>
    <archive>
      <manifestEntries>
        <Premain-Class>com.newrelic.bootstrap.BootstrapAgent</Premain-Class>
        <Can-Redefine-Classes>true</Can-Redefine-Classes>
        <Can-Retransform-Classes>true</Can-Retransform-Classes>
      </manifestEntries>
    </archive>
  </configuration>
</plugin>

Now is coming the tricky part :) NewRelic agent also contains class with main() method which causes that Spring Boot repackager plugin is unable to find single main() method so build fails. It's not a problem but we have to remember to specify proper main class in spring-boot-maven-plugin (or in gradle plugin):
<configuration>
  <mainClass>my.custom.Application</mainClass>
</configuration>

That's all! You can execute your application with following command:

java -javaagent:myapp.jar -jar myapp.jar

Last but not least: don't forget to include NewRelic configuration file (newrelic.yml) in the same directory as your application jar. The other solution is to set newrelic.config.file system property to point the fully qualified file name. 

You can find sample project with proper configuration on my GitHub: https://github.com/jkubrynski/spring-boot-new-relic-agent
