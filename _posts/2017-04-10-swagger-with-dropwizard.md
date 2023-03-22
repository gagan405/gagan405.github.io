---
layout: post
title: Swagger & Dropwizard
subtitle: Not so much of a smooth ride
---

Quite a sad story that `swagger (1.5.x)` does not go so smooth with `Dropwizard (0.9)`, particularly when you have joda `LocalDateTime` objects, authenticated APIs, and probably `snake_case` dto objects.

I spent quite sometime around it to make it work. Here is a summary for later reference:

## Joda LocalDateTime

Joda `LocalDateTime` objects are not serialized to human readable ISO date time formats, and rather they appear as a bunch of objects. This could be overcome in version 1.3 by adding a custom `ModelConverter` in swagger. Unfortunately that was deprecated in version 1.5.

To get this to work, I had to get the Swagger object and iterate through all definitions, and replace the `LocalDateTime` type with proper strong format.

```java
Map<String, Model> definitions = swagger.getDefinitions();
  for(Map.Entry<String, Model> e : definitions.entrySet()){
    Map<String, Property> propertyMap = e.getValue().getProperties();
    for(String key : propertyMap.keySet()){
      Property value = propertyMap.get(key);
      if(value.getType().equals("ref") && ((RefProperty) value).getSimpleRef().equals("LocalDateTime")){
        propertyMap.put(key, new StringProperty("LocalDateTime in ISO format")
                .example("dd-mm-yyyy")
                .pattern("pattern")
                .description("ISO format string"));
      }
    }
  }
```

## Snake Case

Swagger also failed to generate proper example request/response objects if `@JsonSnakeCase` was mentioned. Basically, it just ignored it. Turned out, Swagger was using its own instance of `ObjectMapper`. To get this to work, I did an override of the default `ObjectMapper`.

```java
ObjectMapper converter = new ObjectMapper();
converter.setPropertyNamingStrategy(
    PropertyNamingStrategy.CAMEL_CASE_TO_LOWER_CASE_WITH_UNDERSCORES);
ModelConverters.getInstance().addConverter(new ModelResolver(converter));
```

However, the same object mapper could not be used to generate the `swagger.json` definition file. For the simple reason that we do not want to deserialize the Swagger object in our own way and rather let Swagger do it. So, we use a normal `ObjectMapper` to do it.

```java
ObjectMapper mapper = new ObjectMapper();
mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
String swaggerJson = mapper.writeValueAsString(swagger);
try {
  // Convert object to JSON string and save into a file directly
  mapper.writerWithDefaultPrettyPrinter().writeValue(new File(filePath), swagger);
} catch (JsonGenerationException | JsonMappingException e) {
  e.printStackTrace();
}
return Response.ok(swaggerJson).build();
```

## Authorization Header

In the generated json definition, I needed `Authorization` headers to be sent. Somehow, I couldn't figure out a way to do it with Dropwizard, and I ended up modifying the generated Swagger object to add the Authorization requirement.

```java
Swagger swagger = (Swagger) response.getEntity();
      overrideDateTimeDefinitions(swagger);
      ApiKeyAuthDefinition apiKeyAuthDefinition = new ApiKeyAuthDefinition("authorization", In.HEADER);
      Map<String, SecuritySchemeDefinition> map = new HashMap<>();
      map.put("api_key", apiKeyAuthDefinition);
      swagger.setSecurityDefinitions(map);
      SecurityRequirement requirement = new SecurityRequirement();
      requirement.requirement("api_key", new ArrayList<>());
      swagger.setSecurity(Lists.<SecurityRequirement>newArrayList(requirement)); 
```

After all these changes, it generated the swagger definition json file correctly. I added all these changes to a resource class which extended the default `ApiListingResource`. The code can be found [here](https://github.com/gagan405/swagger_dropwizard_resource/blob/master/SwaggerHelperResource.java).

### Misc

I also added two of those findings as answers on StackOverflow.

* [http://stackoverflow.com/questions/40819084](http://stackoverflow.com/questions/40819084)
* [http://stackoverflow.com/questions/40804899](http://stackoverflow.com/questions/40804899)

   

