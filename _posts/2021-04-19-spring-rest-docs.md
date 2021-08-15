---
layout: single
title: "Spring REST Docs 설정하기"
---

## 개요
Spring REST Docs 는 RESTful 서비스를 문서화 하는데 도움이 됩니다.

- Asciidoctor로 작성된 수기 문서와 Spring MVC 테스트로 생성 된 자동 생성 스니펫을 결합합니다.
- 이 접근 방식을 사용하면 Swagger와 같은 도구로 생성 된 문서의 한계에서 벗어날 수 있습니다.
- Spring REST Docs의 목적은 RESTful 서비스에 대한 정확하고 읽기 쉬운 문서를 생성하도록 돕는 것입니다.

### Asciidoctor
- Asciidoctor는 일반 텍스트를 처리하고 필요에 맞게 스타일이 지정되고 레이아웃 된 HTML을 생성합니다. 원하는 경우 Markdown을 사용하도록 Spring REST Docs를 구성 할 수도 있습니다.
- 스니펫(snippet) : 테스트로 인해 생성된 조각 문서 (이미지 첨부) 


- 이 테스트 기반 접근 방식은 서비스 문서의 정확성을 보장하는 데 도움이됩니다. 스니펫이 올바르지 않으면 이를 생성하는 테스트가 실패합니다. (테스트가 실패한다면 스니펫도 만들어지지 않는다.)
- asciidotor reference: https://asciidoctor.org/

## 요구사항
- Java 8
- Spring Framework 5 (5.0.2 or later)
- 또한 spring-restdocs-restassured 모듈에는 REST Assured 3.0이 필요합니다.

## 빌드 구성
```groovy
plugins { 
	id "org.asciidoctor.convert" version "1.5.9.2"
}

dependencies {
	asciidoctor 'org.springframework.restdocs:spring-restdocs-asciidoctor:{project-version}'
  testImplementation 'org.springframework.restdocs:spring-restdocs-mockmvc:{project-version}' 
}

ext { //생성된 스니펫의 출력 위치를 정의하는 속성을 구성
	snippetsDir = file('build/generated-snippets')
}

test { // 스니펫 디렉터리를 출력으로 추가하도록 test task 을 구성
	outputs.dir snippetsDir
}

asciidoctor { // asciidoctor task -> asciidoctor task 작업 전에 test task가 실행  
	inputs.dir snippetsDir 
	dependsOn test 
}
```

## 문서 Packaging
- 생성 된 문서를 프로젝트의 jar 파일에 패키징 할 수 있습니다.
- 예를 들어 Spring Boot에서 정적 콘텐츠로 제공하도록 할 수 있습니다. 이렇게하려면 다음과 같이 프로젝트 빌드를 구성하세요.
1. jar가 빌드되기 전에 문서가 생성됩니다.
2. 생성 된 문서는 jar에 포함됩니다.
```groovy
bootJar {
	dependsOn asciidoctor // jar를 빌드하기 전에 문서가 생성되었는지 확인하십시오.
	from ("${asciidoctor.outputDir}/html5") {     // 생성 된 문서를 jar의 static / docs 디렉토리에 복사합니다.
		into 'static/docs'
	}
}
```
### 참고 
- Gradle multi module 설정에서 Spring REST Docs설정이 포함된 모듈이 하위 모듈에 속하게 될 경우 jar 설정이 true 일 것이다. 
```groovy
bootJar.enabled = false
jar.enabled = true
```
- 이 경우 위 bootJar Task 는 jar 로 변경이 되어야 정상 동작이 확인가능하다
- 토이 프로젝트(caregiver)에서 오랜시간 동안 삽질을 했다. (기억하자)
```groovy
jar {
    dependsOn asciidoctor
    from("${asciidoctor.outputDir}/html5") {
        into "static/docs"
    }
}
```


## 문서 스니펫 생성
- Spring REST Docs는 Spring MVC의 테스트 프레임 워크, Spring WebFlux의 WebTestClient 또는 REST Assured를 사용하여 **문서화중인 서비스에 요청**을합니다. 
그런 다음 요청 및 결과 응답에 대한 문서 스 니펫을 생성합니다.
  
- RestDocumentationExtension은 프로젝트의 빌드 도구에 따라 출력 디렉터리로 자동 구성됩니다.


### 테스트 설정
### Junit 5 Test Setting
- JUnit 5를 사용할 때 문서 스 니펫을 생성하는 첫 번째 단계는 RestDocumentationExtension을 테스트 클래스에 적용하는 것입니다. 다음 예는이를 수행하는 방법을 보여줍니다.
```java
@ExtendWith(RestDocumentationExtension.class)
public class JUnit5ExampleTests {
  
}

//일반적인 Spring 애플리케이션을 테스트 할 때 SpringExtension도 적용해야합니다.
@ExtendWith({RestDocumentationExtension.class, SpringExtension.class})
public class JUnit5ExampleTests {
  
}
```
- JUnit 5.1을 사용하는 경우 확장을 테스트 클래스의 필드로 등록하고 생성시 출력 디렉토리를 제공하여 기본값을 재정의 할 수 있습니다.
  다음 예는이를 수행하는 방법을 보여줍니다.
```java
public class JUnit5ExampleTests {

	@RegisterExtension
	final RestDocumentationExtension restDocumentation = new RestDocumentationExtension ("custom");

}
```
- 다음으로 @BeforeEach 메서드를 제공하여 MockMvc, WebTestClient 또는 REST Assured를 구성해야합니다. 다음 목록은 그 방법을 보여줍니다.

```java
public class JUnit5ExampleTests {
  private MockMvc mockMvc;
  
  @BeforeEach
  public void setUp(WebApplicationContext webApplicationContext,
                    RestDocumentationContextProvider restDocumentation) {
    this.mockMvc = MockMvcBuilders.webAppContextSetup(webApplicationContext)
      .apply(documentationConfiguration(restDocumentation))
      .build();
  }
}
```
## Mock MVC로 자동 구성된 Spring REST 문서 테스트
- @AutoConfigureRestDocs 애노테이션을 사용하면 REST Docs 코드를 쉽게 추가할 수 있다
- @AutoConfigureRestDocs는 Servlet 기반 웹 애플리케이션을 테스트 할 때 Spring REST Docs를 사용하도록 MockMvc Bean을 사용자 정의합니다.
- @AutoConfigureRestDocs는 기본 출력 디렉토리 (Maven을 사용하는 경우 target / generated-snippets, Gradle을 사용하는 경우 build / generated-snippets)를 재정의하는 데 사용할 수 있습니다.
- 또한 문서화 된 URI에 나타나는 호스트, 체계 및 포트를 구성하는 데 사용할 수도 있습니다.
- @Autowired를 사용하여 삽입하고 다음 예제와 같이 Mock MVC 및 Spring REST Docs를 사용할 때 일반적으로 사용하는 것처럼 테스트에서 사용할 수 있습니다.

```java
import org.junit.jupiter.api.Test;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.restdocs.mockmvc.MockMvcRestDocumentation.document;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(UserController.class)
@AutoConfigureRestDocs
class UserDocumentationTests {

    @Autowired
    private MockMvc mvc;

    @Test
    void listUsers() throws Exception {
        this.mvc.perform(get("/users").accept(MediaType.TEXT_PLAIN))
                .andExpect(status().isOk())
                .andDo(document("list-users"));
    }

}
```
- @AutoConfigureRestDocs의 속성에서 제공하는 것보다 Spring REST Docs 구성을 더 많이 제어해야하는 경우 다음 예제와 같이 RestDocsMockMvcConfigurationCustomizer Bean을 사용할 수 있습니다.
```java
@TestConfiguration
static class CustomizationConfiguration
        implements RestDocsMockMvcConfigurationCustomizer {

    @Override
    public void customize(MockMvcRestDocumentationConfigurer configurer) {
        configurer.snippets().withTemplateFormat(TemplateFormats.markdown());
    }

}
```





















## 참고
- Spring REST Docs Reference : https://docs.spring.io/spring-restdocs/docs/2.0.5.RELEASE/reference/html5/
- Auto-configured Spring REST Docs Tests
  : https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-testing-spring-boot-applications-testing-autoconfigured-rest-docs

