# REST 


## @ResponseBody

* HttpMessageConverter를 이용하면 유저가 요청한 표현형으로 객체를 손쉽게 변환할 수 있다.
> 스프링 MVC가 클래스패스에 있는 것들을 자동 감지하여 JAXB 2, 잭슨. ROME 등의 라이브러리가 발견되면 스프링은 해당 기술에 적합한 HttpMessageConverter를 알아서 등록.
* @ResponseBody는 메서드 실행 결과를 응답 본문으로 취급하겠다고 스프링 MVC에게 밝힘.
* 스프링 4부터는 일일이 메서드에 @ResponseBody를 붙이지 않고 컨트롤러 클래스에 @Controller 대신 @RestController를 붙이는 것이 가능하다.

## @InitBinder

```java
@InitBinder
public void initBinder(WebDataBinder binder) {
    SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");

    binder.registerCustomEditor(Date.class, new CustomDateEditor(dateFormat, false));
}

@RequestMapping("/reservations/{date}")
public void getReservation(@PathVariable("date") Date resDate) {...}
```

## @ResponseEntity

* `ResponseEntity<T>` 는 결과 본문을 HTTP 상태 코드와 함께 집어넣은 래퍼 클래스.

## @RequestMapping의 produces 속성값에 따른 호출
```java
@RequestMapping("/members", produces=MediaType.APPLICATION_XML_VALUE)
@RequestMapping("/members", produces=MediaType.APPLICATION_JSON_VALUE)
```

## RestTemplate 클래스

* REST 서비스를 호출할 의도로 설계된 클래스
```java
RestTemplate restTemplate = new RestTemplate();
String result = restTemplate.getForObject(uri, String.class);
```
> HttpMessageConverter를 활용하여 매핑된 객체를 만듦.

```java
final String uri = "http://localhost:8080/court/member/{memberId}";

Map<String, String> params = new HashMap<>();
params.put("memberId", "1");

RestTemplate restTemplate = new RestTemplate();
String result = restTemplate.getForObject(uri, String.class, params);
```



---

* AtomFeedHttpMessageConverter - Rome Feed 객체를 Atom 피드로 (또는 반대 방향으로) 변환 (미디어 타입은 application/atom-xml) Rome 라이브러리가 클래스패스에 존재하는 경우 등록
* BufferedImageHttpMessageConverter - BufferedImages를 이미지 바이너리 데이터로 (또는 반대 방향으로) 변환
* ByteArrayHttpMessageConverter - 바이트 배열 읽기/쓰기. 모든 미디어 타입(*/*)에서 읽고 application/octet-stream으로 쓴다.
* FormHttpMessageConverter - application/x-www-form-urlencoded의 콘텐츠를 MultiValueMap으로 읽는다. 또한 MultiValueMap을 application/x-www-form-urlencoded로 쓰고 MultiValueMap를 multipartform-data로 쓴다.
* Jaxb2RootElementHttpMessageConverter - JAXB2 애너테이션이 적용된 객체로부터 XML(text/xml 또는 applicatoin/xml)을 (또는 반대 방향으로) 읽고 쓴다. JAXB v2 라이브러리가 클래스패스에 존재하는 경우 등록된다.
* MappingJacksonHttpMessageConverter - 타입이 있는 객체 또는 타입이 없는 HashMap으로부터 JSON을 (또는 반대 방향으로) 읽고 쓴다. Jackson JSON 라이브러리가 클래스패스에 존재하는 경우 등록된다.
* MappingJackson2HttpMessageConverter - Jackson2JSON 라이브러리가 클래스패스에 존재하는 경우 등록된다.
* MarshallingHttpMessageConverter - 주입된 마샬러(marshaller)와 언마샬러(unmarshaller)를 이용해 XML을 읽고 쓴다. 지원되는 마샬러(언마샬러)는 Castor, JAXB2, JIBX, XMLBeans 그리고 XStream이 있다.
* ResourceHttpMessageConverter - org.springframework.core.io.Resource를 읽고 쓴다.
* RssChannelHttpMessageConverter - Rome Channel 객체로부터 RSS 피드를 (또는 반대 방향으로) 읽고 쓴다. Rome 라이브러리가 클래스패스에 존재하는 경우 등록된다.
SourceHttpMessageConverter - javax.xml.transform.Source 객체로부터 XML을 (또는 그 반대 방향으로) 읽거나 쓴다.
* StringHttpMessageConverter - 모든 미디어 타입(*/*)을 String 으로 읽는다. text/plain에 대한 String을 쓴다.
* XmlAwareFormHttpMessageConverter - FormHttpMessageConverter를 상속받아 SourceHttpMessageConverter를 이용해 XML 기반의 부분에 대한 지원을 추가한다.
예를 들어, 클라이언트가 요청의 Accept 헤더를 통해 application/json을 받을 수 있음을 나타낸다고 가정하자. Jackson JSON 라이브러리가 애플리케이션의 클래스패스에 있다고 가정하면, 핸들러 메소드에서 반환된 객체는 클라이언트에 반환되는 JSON 표현으로의 변환을 위해 MappingJacksonHttpMessageConverter에 할당된다. 반면에 요청 헤더가 클라이언트는 text/xml을 선호한다고 나타내면, Jaxb2RootElementHttpMessageConverter가 클라이언트에 대한 XML 응답을 생성하는 작업을 수행한다.
---