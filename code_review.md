# CODE REVIEW

# Código

## Clean Code

1. **Legibilidad y productividad**

        Ejemplos
        - Ad.java
        - Picture.java
        - Quality.java
        - Typology.java
        - PublicAdd.java
        - QualityAdd.java
        - AdVO.java
        - PictureVO.java

    En todos los casos se repite la metodología tradicional para creación de métodos GET, SET, toString...

    Existen librerías como **Lombok**, que simplifican toda estas tareas, haciendo el código mucho más legible y agilizando las tareas de desarrollo.

2. **Nombres con sentido**

    Todos los nombres que se utilizan son bastante descriptivos.

3. **Comentarios**

    Faltan comentarios más descriptivos a lo largo de todo el código que ayuden a entender tanto lo que hace como lo que se pretende llevar a cabo.

    Por otro lado tampoco hay ningún uso de Javadoc.

## Buenas prácticas

1. **Versionado**

        Ejemplos:
        - AdsController.java

    En este código ejemplo no se ha tenido en cuenta la versión de la API en la definición del endpoint, puede llegar a ser interesante seguir esta estrategia si, por ejemplo, se quiere utilizar dos versiones a la vez.

2. **Auth**

    No se utiliza ningún tipo de control de autorización. Un uso sencillo podría ser por ejemplo un token de tipo JWT.

3. **Documentación de la API**

    No se documenta/define ninguno de los endpoints, como por ejemplo mediante uso de Swagger o similar.

    Swagger indica los recursos disponibles en la API REST y las operaciones que pueden llamar.

## Eficiencia

1. **Bucles para reescribir objetos**

        Ejemplos
        - AdsServicesImpl#findPublicAds
        - AdsServicesImpl#findQualityAds

    En ambos métodos se recorre un lista completa de resultados para volver a crear una lista de objetos similares, perjudicando al rendimiento en mi opinión.

    Como alternativa se podría definir un serializador y personalizar el retorno (Ej: [enlace a documentación externa - JACKSON](https://www.baeldung.com/jackson-custom-serialization)).

    Si por necesidades del negocio, se declara que **es necesario** la declaración de este nuevo objeto, para obtener un código mucho más limpio, en lugar de setear cada uno de los valores dentro del bucle, crearía un constructor en la clase del nuevo objeto que reciba como parámetro el original:

    ```java
    public PublicAd(Ad ad) {
        this.description = ad.getDescription();
        ...
    }
    ```
    Teniendo este método, incluso podemos refactorizar aún más nuestro código, por ejemplo:

    ```java
    List<PublicAd> result = ads.stream().map(x -> new PublicAd(x)).collect(Collectors.toList());
    ```

    Reduciendo todo el código en una solo línea y siendo mucho más legible en mi opinión.

    Existe una variante muy similar en la clase InMemoryPersistence. Para los métodos *mapToDomain* podríamos crear un constructor personalizado en cada uno de los objetos para no tener que abusar de los *get* y pasarle a este constructor el objeto completo, haciendo así el código mucho más legible:

    ```java
    private Ad mapToDomain(AdVO adVO) {
        return new Ad(adVO.getId(),
                Typology.valueOf(adVO.getTypology()),
                adVO.getDescription(),
                adVO.getPictures().stream().map(this::mapToDomain).collect(Collectors.toList()),
                adVO.getHouseSize(),
                adVO.getGardenSize(),
                adVO.getScore(),
                adVO.getIrrelevantSince());
    }
    ```

    Podría reducirse simplemente a:

    ```java
    private Ad mapToDomain(AdVO adVO) {
        return new Ad(adVO);
    }
    ```

2. **Abuso de if-else**

        Ejemplo:
        AdsServicesImpl#calculateScore

    El código debería ser mucho más atómico, para cada uno de los requisitos de negocio que se indican se debería desaclopar, de esta forma en un futuro sería mucho más fácil incorporar nuevos requisitos.

    Por ejemplo podría crearse una clase llamada *Score* la cual, tendría como único atributo la puntuación. A esta clase se pueden ir incorporando diferentes métodos que realicen por separado cada una de los cálculos, por ejemplo *getScorePhotos*, *getScoreDescription*... Por último existirá un método orquestador encargado de ejecutar cada uno de estos cálculos.

    Si seguimos esta estrategia, además de que obtendremos un código mucho más limpio, será más fácil de mantener, testear y añadir nuevas condiciones por la atomicidad que estamos dándole.

3. **Query**

        Ejemplo:
        InMemoryPersistence#findRelevantAds
        InMemoryPersistence#findIrrelevantAds

    ```java
    @Override
    public List<Ad> findRelevantAds() {
        return ads
                .stream()
                .filter(x -> checkValidAddScore(x) && x.getScore() >= Constants.FORTY)
                .map(this::mapToDomain)
                .collect(Collectors.toList());
    }
    ```

    En ambos casos se accede al stream completo y se filtra. Supongo que en un entorno real se realizará esta consulta creándola directamente sobre la base de datos (p.e.: query sql).

## Errores

1. **PALABRAS PARECIDAS - AdsServicesImpl#calculateScore**

    ```java
            if (wds.contains("luminoso")) score += Constants.FIVE;
            if (wds.contains("nuevo")) score += Constants.FIVE;
            if (wds.contains("céntrico")) score += Constants.FIVE;
            if (wds.contains("reformado")) score += Constants.FIVE;
            if (wds.contains("ático")) score += Constants.FIVE;
    ```

    No me parece correcto la forma en la que se comprueba si la palabra forma parte de la descripción. Deberíamos tener en cuenta un porcentaje de error. Puede ser que el usuario escriba sin alguna tilde, o que por error falte o sobre alguna letra en la palabra. Por lo que crearía una evaluación que tuviese en cuenta estas condiciones.

2. **CÁLCULO COMPLETIDUD - AdsServicesImpl#calculateScore**

    ```java
        //Calcular puntuación por completitud
        if (ad.isComplete()) {
            score = Constants.FORTY;
        }
    ```
    Se esta pisando el valor anterior *score* por lo que supongo que se pretendía utilizar, en su lugar:

    > score += Constants.FORTY;

3. **SETEO DE PUNTUACIÓN - AdsServicesImpl#calculateScore**

    ```java
            if (Typology.FLAT.equals(ad.getTypology())) {
                if (wds.size() >= Constants.TWENTY && wds.size() <= Constants.FORTY_NINE) {
                   score += Constants.TEN;
                }

                if (wds.size() >= Constants.FIFTY) {
                    score += Constants.THIRTY;
                }
            }
    ```

    **Según la historia de usuario:**  
    
    > ... En el caso de los pisos, la descripción aporta 10 puntos si tiene entre 20 y 49 palabras o 30 puntos si tiene 50 o más palabras... 

    Tal y como esta diseñado, si una descripción tiene 50 palabras se sumarían 40 puntos en lugar de 30, para solucionarlo sería una claúsula *if - else* en lugar de dos *if*.

4. **Date vs LocalDateTime - AdsServicesImpl#calculateScore**

    ```java
        if (ad.getScore() < Constants.FORTY) {
            ad.setIrrelevantSince(new Date());
        ...
    ```

    Existen multitud de recomendaciones de hacer este cambio a partir de la versión 1.8 de java. Consideran el uso de Date como deprecado.

5. **ENDPOINT RESPONSE - Casos de error**

    Se da por hecho que todas las respuestas son correctas pero existen muchos más casos a parte de 200OK: 2xx, 3xx, 4xx y 5xx.

    Un claro ejemplo puede ser el endpoint *score*. Puede que haya ocurrido algún tipo de error, pero el usuario recibirá una respuesta 200OK.

6. **NullPointer - InMermoryPersistence#findIrrelevantAds & #findRelevantAds**

    ```java
    @Override
    public List<Ad> findIrrelevantAds() {
        return ads
                .stream()
                /**
                 * Hay que tener en cuenta valores nulos para evitar algún tipo de NullPointer
                 */
                .filter(x -> checkValidAddScore(x) && x.getScore() < Constants.FORTY)
                .map(this::mapToDomain)
                .collect(Collectors.toList());
    }
    ```

    El valor *x.getScore()* puede ser nulo, por lo que tenemos que tenerlo en cuenta.

    Este error se puede aparecer en todos los accesos get definidos en la API en el caso de que por dominio no se obligue a que estos objetos no puedan tener atributos nulos (@NoNull)

    ```java
    private Boolean checkValidAddScore(AdVO adVO) {
        return adVO != null && adVO.getScore() != null;
    }
    ```

    Con el método anterior por ejemplo podríamos evitar encontrarnos con este problema.

7. **SAVE VS UPDATE**

        Ejemplos:
        InMemoryPersistence#save(Ad ad)
        InMemoryPersistence#save(Picture picture)

    En ambos casos sigue una estructura similar:

    ```java
    @Override
    public void save(Ad ad) {
        ads.removeIf(x -> x.getId().equals(ad.getId()));
        ads.add(mapToPersistence(ad));

        ad.getPictures()
            .forEach(this::save);
    }
    ```

    ¿Realmente el comportamiento que se quiere si tiene el mismo ID es eliminar/sobreescribir el anterior?
    Parece más un update que un save.

# Testing

 1. No se ha usado TDD.
 2. No se comprueban realmente resultados esperados para todos los métodos.
 2. No se comprueban los métodos en sí, solo uno del servicio.
 3. La cobertura de clases se queda en el 38% y de líneas en 22%. La de líneas al menos deberían llegar el 70%.
 4. Solo se hacen test unitarios y no de integración.
 5. Las pruebas deben seguir la misma estructura que el código base, ayudando a comprender mejor el código.

# Resumen

Para refactorizar el código, mi primera idea sería desaclopar al máximo toda su estructura, además añadir más pruebas al código, también centradas en algunas partes unitarias del código, ayudará a encontrar algunos puntos débiles en el código (no sólo funcionales sino de diseño)