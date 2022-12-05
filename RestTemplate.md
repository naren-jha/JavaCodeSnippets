
To make an HTTP request in Java with Spring Boot, you can use the RestTemplate class. Here is an example:

```Java
// Create a new RestTemplate instance
RestTemplate restTemplate = new RestTemplate();

// Make the HTTP GET request to the specified URL
ResponseEntity<String> response = restTemplate.getForEntity("http://www.example.com", String.class);

// Print the response body to the console
System.out.println(response.getBody());
```

Alternatively, you can use the WebClient class, which is part of the Spring WebFlux framework. Here is an example of how to use WebClient to make an HTTP GET request:

```Java
// Create a new WebClient instance
WebClient client = WebClient.create();

// Make the HTTP GET request to the specified URL
Mono<String> response = client.get().uri("http://www.example.com").retrieve().bodyToMono(String.class);

// Print the response body to the console
response.subscribe(System.out::println);

```
