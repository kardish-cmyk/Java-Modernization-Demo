# Migration notes: Java 8 / Spring Boot 2.7 to Java 21 / Spring Boot 3.5

For engineers maintaining this application, and for teams planning the same upgrade in other services.

## What changed

| Area | Before | After |
|---|---|---|
| Java | 8 | 21 |
| Spring Boot | 2.7.3 | 3.5.16 |
| Spring Framework | 5.3 | 6.2 |
| Hibernate | 5.6 | 6.6 |
| Servlet container | Tomcat 9 | Tomcat 10.1 |
| Enterprise APIs | Java EE (`javax.*`) | Jakarta EE 10 (`jakarta.*`) |

- **Imports**: `javax.persistence`, `javax.validation` and `javax.xml.bind` became `jakarta.persistence`, `jakarta.validation` and `jakarta.xml.bind`. `javax.cache` (JCache) is unchanged because that API was never moved to `jakarta`.
- **Dependencies**:
  - MySQL driver is now `com.mysql:mysql-connector-j`.
  - Ehcache uses the `jakarta` classifier.
  - `jakarta.xml.bind-api` is declared explicitly, because JAXB is no longer part of the JDK.
- **Build plugins**: JaCoCo, Checkstyle and spring-javaformat were upgraded to versions that understand Java 21, and the git-commit-id plugin moved to its new coordinates. The newer formatter re-wrapped some chained method calls; those hunks contain no logic changes.
- **Deprecated APIs**: tests use `@MockitoBean` instead of `@MockBean`, and `SerializationUtils.deserialize` was replaced with `ObjectInputStream`.
- **Tooling**: CI builds on Temurin 21, the devcontainer uses a JDK 21 image, and `docker-compose.yml` runs MySQL 8.4 instead of 5.7.
- **Artifact**: the jar is now `spring-petclinic-3.5.16.jar`.

Application behaviour is otherwise unchanged. The test suite (41 tests) passes before and after, with line coverage at 96.30% before and 96.36% after.

## How to run on Java 21

Prerequisite: a JDK 21 on `JAVA_HOME` (any distribution; CI uses Temurin). Maven is provided by the wrapper.

```bash
java -version                 # must report 21
./mvnw verify                 # build, tests, coverage report in target/site/jacoco/index.html
java -jar target/spring-petclinic-3.5.16.jar
```

The app listens on port 8080 and uses an in-memory H2 database by default.

To run against MySQL or PostgreSQL:

```bash
docker compose up -d mysql    # or: postgres
java -jar target/spring-petclinic-3.5.16.jar --spring.profiles.active=mysql   # or: postgres
```

If you have a local MySQL 5.7 container from before the upgrade, recreate it (`docker compose down -v`); changing the image does not upgrade existing data.

The build enforces formatting. If `./mvnw verify` fails in the `validate` phase after you edit code, run `./mvnw spring-javaformat:apply`.

## Known behaviour change in this application

URLs with a trailing slash no longer match. `/owners/find/` returned 200 on Spring Boot 2 and returns 404 now; `/owners/find` is unaffected. Update any bookmarks, links, monitors or client calls that use a trailing slash.

## Risks to check in other services

The items below either surfaced in this upgrade or are documented Spring Boot 3 changes that this small application did not exercise. The second group is marked as not exercised here.

**Seen in this upgrade**

- **Runtime**: Spring Boot 3 needs Java 17 or later, and a service compiled for Java 21 (as this one is) needs a Java 21 runtime. Check base images, deployment hosts, CI runners and any build tooling pinned to Java 8 or 11.
- **`javax` to `jakarta`**: every dependency that touches servlet, JPA, validation, JAXB, mail or JMS types must have a Jakarta-compatible version. Look for libraries that need a `jakarta` classifier (as Ehcache did) and for in-house shared libraries compiled against `javax.*`. Do not blanket-replace `javax.`: packages that ship with the JDK (`javax.sql`, `javax.crypto`, `javax.net`, `javax.naming`) and `javax.cache` stay as they are.
- **APIs removed from the JDK**: JAXB, JAX-WS and `javax.annotation` were bundled with Java 8 and are gone. Code that compiled without declaring them needs explicit dependencies.
- **Trailing-slash matching**: off by default in Spring 6. Check controllers, gateway routes, health-check URLs and API clients.
- **Hibernate 6**:
  - Minimum database versions were raised. MySQL 5.7 still connects but is logged as unsupported.
  - Generated SQL, sequence and ID generation defaults, and some type mappings changed.
  - Test services against a copy of their real schema and data, not only against H2. This application was checked against its sample schema on H2, PostgreSQL 14 and MySQL 8.4.
- **Relocated artifacts**: dependencies and plugins that Spring Boot no longer manages under their old coordinates fail with a missing-version error (here, the MySQL driver and the git-commit-id plugin).
- **Build properties**: Boot 3 compiles with `--release`, so `maven.compiler.source` and `maven.compiler.target` are no longer set. Anything referencing them, as the `build-info` goal did here, must use `java.version`.
- **Build tool versions**: older JaCoCo, Checkstyle and formatter plugins fail on Java 21 class files or syntax.
- **Test annotations**: `@MockBean` and `@SpyBean` are deprecated for removal; move to `@MockitoBean` and `@MockitoSpyBean`.

**Not exercised here, check where relevant**

- **Spring Security**: Boot 3 brings Spring Security 6, which removes `WebSecurityConfigurerAdapter` and `antMatchers` and changes several defaults. This application has no security configuration, so none of that was tested.
- **Configuration properties**: a number of `spring.*` and `management.*` keys were renamed between 2.7 and 3.x. Adding `spring-boot-properties-migrator` temporarily reports them at startup. None of this application's properties needed changing.
- **Auto-configuration registration**: libraries that register auto-configuration through `META-INF/spring.factories` must move to `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
- **HTTP clients and observability**: `RestTemplate` on Apache HttpClient needs HttpClient 5, and Spring Cloud Sleuth is replaced by Micrometer Tracing.
- **Actuator exposure**: services that expose all endpoints (`management.endpoints.web.exposure.include=*`, as this one does) also expose endpoints added in newer versions.
- **Libraries using reflection or bytecode generation**: older versions of Lombok, MapStruct, Mockito, ByteBuddy and similar may not support Java 21 and should be upgraded with the framework.

## Suggested order for other services

1. Upgrade to the latest Spring Boot 2.7.x on the current JDK and clear deprecation warnings.
2. Move the build and runtime to Java 17 or 21 while still on Boot 2.7.
3. Upgrade to Spring Boot 3 and migrate `javax` to `jakarta`.
4. Run the full test suite, then test against a production-like database and exercise the HTTP surface, including trailing-slash URLs.
