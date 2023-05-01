# DDD en TypeScript: Modelado y arquitectura

## Init
-- Clonamos el Repo
  - Todo esta opiniondo para aplicar DDD logrando una alta escalabilidad.


-- Tests
  - Test de Caja Negra:   Cucumber
  - Unit Test:            Jest


-- Deps:
  - Express.js
  - express-promise-router: Ayuda a tener los Controller mas limpios SIN necesidad de usar Callbacks
  - helmet: seguridad
  - winston: tema de Logs
  - husky:
    - Permite registrar hooks q se van a ejecutar cada vez q hagas lo q le indiques
      - En este caso c/cvez q se haga  commit  (git)
        - Ejecutara lo definido en el      'lint-staged'
          - Asi con cada commit formatea el codigo, corre el linter, etc.












## 👩‍⚕️ Health check de la aplicacion: Nuestro primer endpoint

### 🔚📍Crear Endpoint de health check: Controllers asíncronos con Express y declaracion de rutas dinámica

-- Iniciamos con el Endpoint q nos dira si se esta levantando el server/API para la APP backend del BC de Mooc
  - Creamos el    `StatusGetController.ts`
    - Creamos la Class      'StatusGetController'
      - Podemos ser mas verbosos y nombrarla como   StatusGetHttpController
        - Pero por Convencion de Equipo de Desarrollo asociamos todos los verbos HTTP a un HTTP, asi que solo se queda Get
  - Es importante que este   endpoint    SOLO valide que se levanta el Server
    - NOO debe validar la conexion a DB xq
      - Xq si mato este servidor q me lanza un 500 xq la DB esta muerta/caida, pues con AutoScale voy a matar este Server y voy a levantar otro, pero eso daria un loop de errores xq la DB segueria caida.



-- Inyeccion de Dependencias:
  - Usan   `node-dependency-injection`   que se basa en ficheros YML
    - Donde el    `const loader = new YamlFileLoader(container);`
      - BUSCA el      'application.yaml'
        - Que tiene todos los   `resource`    a otros YMLs con sus Services que vienen a ser    `NamesPace`   para usarlos en la inyeccion de dependencias. En este caso tiene de    resource a   './app/application.yaml'  que ya tiene sus namespaces para usarlos:
```yml
services:

  Apps.mooc.controllers.StatusGetController:
    class: ../../controllers/StatusGetController
    arguments: []
```
        - Aqui el Namespace es el de    Apps.mooc...   |  Class es la referencia al fichero de la Class q vamos a inyectar
          - Y si tiene argumentos los pasaremos en el array, pero ojo, SIN Acoplar el Codigo
            - Ej: Class Logger  y NO la libreria externa directamente
        - La inyeccion de dependencias sera asi:

```ts
// namespace
const controller: StatusGetController = container.get('Apps.mooc.controllers.StatusGetController');
```



### ✅ Añadir Integracion Continua: Tests de Aceptacion
-- Usamos    Github Actions    para que valide los tests al hacer commit
  - Meter Integracion Continua con GitHub Actions para que todos los Test que se agreguen se ejecuten al hacer commit
    - Aseguramos que los cambios que se metan a futuro NO rompan nada de lo ya existente y testeado
      - Estas buenas practicas desde el inicio valen oro
        - Asi no metemos nada q pueda romper la app
  - Al Empezar un Project una de las cosas en las que mas merece la pena Invertir Tiempo es en Sentar esas Bases de Buenas Practicas de la PipeLine
    - Y hacer estos tests con CI


-- Todos los tests de     `App`    van a ser tratados como     `Tests de Aceptacion / tests de caja negra`
  - Replicamos la Misma estructura del codigo de produccion en los tests

  -- Los archivos   `.feature`   don Lenguaje Gherkin
        Each step starts with Given, When, Then, And, or But:
          Examples follow this same pattern:
            Describe an initial context (Given steps)
            Describe an Event (When steps)
            Describe an expected outcome (Then steps)

  -- Estos spteps del Lenguaje Gherkin debemos traducirlos a tests
    - Para eso tenemos los    /spet_definition/*.steps.test
      - El test levanta la app y luego la para. Esto ya es diferente a lo normal q se hace es matar la app
        - BeforeAll:  Antes de c/test va a levantar la app
        - AfterAll:   paramos la app

    - Con Gherkin definimos los steps del test
      - Los llevamos a codigo en este caso para el   'controller.steps.ts'
        - A este codigo le dicemos de la existencia de Cucumber con el   `cucumber.js`
          - Aqui agrupamos el modulo para testear q va a ser usado en los   Scripts   del package.json
          - Por ejemplo para q solo testee las   fetaures   de ese conjunto
            module.exports = {`mooc_backend`};    <--   cucumber.js
            `test:mooc:backend:features": "NODE_ENV=test cucumber-js -p mooc_backend",`

      - Ahora, para el CI o  `Contonuous Integration`  usamos el   `nodejs.yml`  en la carpeta del .github
        - Aqui corremos los tests de aceptacion y los que hayan
          - Hacemos la      build     para asegurnos de que haga el build correctamente
            - Y que NO se nos fue algun herro de typado y demas propio de TS
          - Corremos el    linter    para q tb valide eso
          - Corremos los    tests    propiamente


      https://cucumber.io/docs/gherkin/reference/















## ♻️ Desarrollo Outside-in: Implementacion del caso de uso para crear curso

### ☝️ Implementacion del endpoint y test de Aceptacion
-- Desarrollar la 1ra funcionalidad con toda la base    <--    `Create New Course`
  - Usaremos    `ATDD`     q es el Ciclo q Envuelve a TDD
    - Iniciamos definiendo la funcionalidad/feature desde el punto de vista del Client q lo va a Consumir
      - Crearesmo el     endpoint     de CreateCourse, en donde el Client es el q consueme la API

  -- Creamos el Test de Aceptacion
    - 1ro Gherkin:    /mooc/backend/features/courses  <-  `create-course.feature`
      - Ubicarlo es intuitivo xq tenmos 2 carpetas (/src /tests)
        - Es un Test, asi q va en    /tests
          - Es un Test de una APP (backend)   <--   `Test de Aceptacion`
            - Sabemos q es de mooc y su back y es un Test de Aceptacion
              - Como Interacciona x los Entripoints

    -- Crear un Curso: Hacemos 1     `PUT`      xq es el    Client    quien define el   ID    del curso
      - En el  Body  lo q envio es el name y duration
      - Then: Esperamos 1 response solo con el   201
      - AND: que NO tenga cuerpo de respuesta


  -- Create course
    - 1ro se escriben los Tests de Aceptacion en `L. Gherkin` xq es a una App (back/entripoints)
    - 2do creamos los test en CODIGO    <--   controller.steps.ts
      - El ID lo provee el Client
        - Aislandonos de la DB donde el ID sea 1 AutoIncremental definido x la Infrastructure, sino q pasamos 1 UUID
    - Escribimos el codigo
      - Comenzamos x el   `Controller`   q   Implements   la interface de   Controller
      - Creamos el     `Router`   , Inyectando la dependencia del Controller
        - courses.route.ts
        - Como son   `Tests de Aceptacion`   NO debo conocer la infrastructure (DB) xq si eso cambia, todos nuestros tests van a fallar.
          - Debemos enfocarnos en estos tests a lo q es visible para 1 Client
            - Definir cual va a ser el Client, es el q consume el servicio HTTP, el q verifica que se haya creado en la Queue, etc.
      - Lo q falta es el ciclo del tests:
        - Poner en rojo el Unit Test, ponerlo a verde y ver como sale el Test de Aceptacion
        - Aqui ya hemos puesto en Verde lo de afuera del ciclo de UnitTests (ATDD)
          - Ahora debemos meternos con los UnitTests para que a consecuencia de poner en Rojo los UnitTest y pasarlos a verde, ver como queda este Test de Aceptacion externo q rodea el ciclo
    - Refactorizamos
      - Es mas facil xq ya tenemos tests q nos dan seguridad de q lo hecho sigue funcionando.








### 🔥 Implementacion del caso de uso y test unitario
-- Ya tenemos Nuestro Test de Aceptacion  (ATDD)
  - Ahora Empieza     `TDD`      puro con los    `Unit Tests`
    - Como ya vamos a testear      Casos de Uso (logica de negocios)     ya debemos empezar a escribir codigo escalable.
    - Los Unit Tests debe ser independiento del    Protocolo de Comunicacion    q se este exponiendo
      - Como ya vamos a crear el Caso de Uso en el BC y Module especifico
        - Ahora podemos llamar a ese caso de uso desde donde queramos
          - Desde el Controller de la APP Back
          - Ejecutar el Caso de Uso de 'Crear Curso' a consecuencia de 1 Evneto
            - Si se crea curso en Backoffice, tambien crealo en Mooc. (Event Suscriber)

  - Vamos a usar Jest para los UnitTest
    - Pero como usamos TS, necesitamos 1 Preset para q nos indique los errores en TS y no en JS
      - Asi q configuramos el preset en  `jest.config.js`
        - preset: 'ts-jest'
          - Nos va a ayudar a ver problemas de compilacion asi como a ver problemas directamente en la consola en TS



  - -- Como Aqui ya hacemos     TDD     iniciamos con los unit tests
    - Como ya es 1     `CASO DE USO`     esto ya va en el        Contex/`Bounded Context`/BC
      - Xq va a ser un BC va en      `Context/Mooc/Courses/appication`
        - Context xq es un BC
        - Mooc xq es el BC de Mooc
          - Courses           <-    Module
            - application     <-    Arquitectura Hexagonal (Casos de Uso)


    -- /src/Context/Mooc/Courses/appication
      - Creamos el       `CourseCreator.ts`   <--  1ro creamos el test (TDD)
      - De una creamos el homologo en   tests:      /tests/Context/Mooc/Courses/application/`CourseCreator.test.ts`
        - Como estos son      `Unit Tests`    decidimos identificarlos como    `.test.ts`
        - Aqui   `Instanciamos`   el Caso de Uso 'CourseCreator' y NOO se lo pedimos al Contenedor de Inyeccion de Dependencias
          - Estamos  Instanciado   al sujeto q se encuentra bajo Test
          - Le pasamos lo q necesita para el Test (id, name, duration)
            - Esto  NOO  nos lo Inventamos, esto viene del  `ATDD`  q ya generamos
              - En ese ATDD ya establecimos el Contrato de lo q se le envia en el Body
                - Aqui se lo aplicamos
      - Aqui queremos Testear q este Caso de Uso produce un `Side Effect`, q guarda en DB
        - NO enviamos nada en la Response de este caso de uso
        - X eso el Caso de Uso Return una Promise<void>
          - Con lo cual, la razon de existir de este Caso de Uso es q NO devuelva nada y al no devolver nada, PRODUZCA un efecto colateral o SideEffect
            - Con lo cual, en este caso el tipo VOID es el q mas info nos da

        - Si esto NO retorna nada aqui es donde se evidencia la 2da indireccion q intorducimos `Repository`
          - Aqui ya nos indica q el   CourseCreator    Necesita 1 Repository q sea quien asuma la responsabilidad de guardar el course en DB
          - Entonces el   `CourseCreator`  va a tener q  MANEJAR ese Paso de Mensajes e instanciar los Elementos de Dominio q correspondan y PASAR el MENSAJE al q Tenga la Responsabilidad de Almacenarlos (Repository)
            - Aqui en este Test vamos a Testear el   PASO de MENSAJES
              - Vamos a Testear como COLABORAN estos 2 elementos
                - CourseCreator: Sujeto bajo Test
                - La barrera q pongo para No contamianr los tests con lo q use para manejar la DB (infrastructure)
                  - Como si uso 1 JSON o redis, aqui no me importa xq lo q me interesa testear ahora es la  `Logica de negocio`
          - Todos los    IMPORTS   en el Caso de uso (CourseCreator) SIEMPRE deben ser de   `DOMAIN`
            - NUNCA deberia haber 1 import de infrastructure
              - Ya q al hacer eso violamos la ley de dependencia de la Arq. Hexagonal
                - Donde todas las capas mas externas SOLO deben Depender de las capas mas Internas
                  - infrastructure > Application > Domain    <- Arq. Hexagonal
                    - Asi, si Cambiamos algo de infrastructure, NO tenemos q cambiar NADA de las capas Internas

          - CREAMOS el   `CourseRepository.ts`    <--  INTERFACE
            - Lo creamos en   /src/Context/Mooc/Courses/domain
              - Es    CourseRepository    es un Contrato de DOMINIO, de la Capa q Nosotros SI que controlamos
                - Independientemente de las particularidades de la implementacion en concreto (infrastructure - librerias)
              - Aqui NO deberiamos estar recibiendo los Distintos Elementos q Componen un Course y componerlo
                - Xq eso querria decir q la Responsabilidad de Instanciar 1 Course la Estamos Delegando en al infrastructure
                  - Y si cambia esa pieza de infrastructure, vamos a tener q duplicar la logica de como Instanciar el Course

          - Entonces, el CASO DE USO  CourseCreator, lo que va a ser es:
            - Recibir la Informacion q compone el Elemento de Dominio
              - Todo lo q compone un Course (id, name, duration)
            - Instanciar el Elemento de Dominio (Course)
            - Y pasar el mensaje a su Colaborados (CourseRepository)

        - Ahora q ya tenemos implementado todo esto, lo q hacemos es correr los UnitTests
          - El comando es     `npm run test:unit`








### 🙆‍♂️ Implementacion del repositorio en fichero y test de integracion
- --- En este punto YA tenemos el Controller x 1 lado, el Caso de Uso x otro
  - Falta la Implementacions el Repository y el UnitTest perteneciente a ese Repository para ver como nos integramos

  - Aqui al tener ya los UnitTest, estamos en el N Cycle de UnitTests q a su vez es englobado x el Test de Aceptacion (ATDD)
    - Ahora lo q viene es colocar en Verde todos los UnitTest
    - Una vez los coloquemos en Verde procedemos a salir hacia afuera del N Cycle para poner en Verde el Test de Aceptacion



  - -- Nos habiamos quedado con un Controller q no hace nada, y ahora q YA tenemos el Caso de Uso Implementado procedemos a Implementar el Controller   (`CoursePutController`)   <--  /src/app/.../backend/controllers
    - Este Controller de momento tiene que Llamar al Caso de Uso
      - De momento NOO estamos Metiendo el  Command Bus de CQRS, esto lo veremos en otro curso
        - De momento Lo q QUeremos es esa Modularidad de DDD sin CQRS para q vosotros podais evaluar q es lo mejor q les conviene

    -- Aqui la  COLABORACION  la hacemos a traves de la     `Dependency Injection`
      - Entonces, aqui en el Controller No hace falta 1 Try/Catch  xq  tenemos un Middleware
        - Q en caso de error inesperado, como q falla el Caso de Uso xq se cae DB, pues se lanza un 500
      - Si vamos a inyectar el CourseCreator en el Controller debemos actualizar el Container de Dependency Injection


    -- Creamos el test     `FileCourseRepository.test.ts`   replicando la jerarquia de carpetas de Prod
      - Esperamos q la Entidad de DOMAIN q nos devuelve el    .search()    sea la misma q almacenamos
      - Como es TDD, ya tenemso el Unit Test, asi q ahora Implementamos el COdigo para q pase el test


    -- La Inyeccion de Dependencias con el de    'node-dependency-injection'
      - Creamos el   /src/apps/mooc/backend/dependency-injection/`Courses/application.yaml`
      - Creamos 1 Nuevo `namespace` q sea el de  'Mooc.Courses.domain.CourseRepository'
        - Com la Intrface de CourseRepository es de DOMAIN la dejamos con el namespace .domain
          - Pero Apunta a la Implementacion de infrastructure  'FileCourseRepository'
        - Y este namespace lo pasaremos como   `arguments`  al namespace de   'CourseCreator'
      - Ahoa si q podemos agregarselo al   'Apps.mooc.controllers.CoursePutController'  q esta en
        - /src/apps/mooc/backend/dependency-injection/apps/application.yaml
        - Aqui asociamos el   `arguments: ["@Mooc.Courses.application.CourseCreator"]`   creado antes















## 💪 Refactorizando aprovechando el potencial de TypeScript
### 🔀 Mocks mas semanticos y mantenibles
- --- Podemos hilar fino con los tests y mejorarlos

  - -- Vamos a mejorar el     Test del Caso de Uso (CourseCreator)    con mocks y demas
    -- Creamos el    `CourseRepositoryMock.ts`   en /tests/...Mooc/../__mocs__
      - En esta Class tambien creamos una    jest.fn()   pero la enclobamos dentro del   save   q hereda del Repository
        - Esto es mas seguro, pues asi sabemos q en efecto se llamo el   save()   de CourseRepository, ya q podemos saber cuantas veces se llamo las   jest.fn()
          - Asi registramos 1 llamada al    save()    con los Parameters q son
            - X c/d Method del Repository vamos a tener un mock para saver q ha sido llamado con los parameters q se necesitan

    - Aqui todo queda mucho mas semantico
      - Primero la creacion del course
      - La creacion, en donde es el Caso de Uso quien Instancia el Course
        `await creator.run(id, name, duration);`
          - X eso es q Instaciamos en el Casoo de Uso y NO lo Inyectamos
            - No todo es inyeccion, en estos casos Instanciar manualmente nos Facilita la Vida con los Tests
      - Luego hacemos la Acersion
        - Y como tenemos armado el  `CourseRepositoryMock`   podemos asegurar q se va a llamar con lo q se necesita q se llame.
          - Si da error, ya NO nos muesta solo el boolean
            - Compara lo q Instancia el Caso de Uso con el   'expectedCourse'
              - Y esto, si da discrepancias, nos indica exactamente lo q fallo y no solo un boolean






### 🙋 Validacion de requests: Los tipos son tus amigos
- --- Hasta este punto tenemos un puerto espueto en nuestra API HTTP
  - En el Controller estamos recibiendo data, pero NO la estamos validando, asi q si nos envian un DataType ajeno al q esperamos, esto peta.
    - Pero peta rarisimo y NO controlado ya q caeria al    Middleware    q lanza todo tipo de excepcion con un  500
      - Esto es un Error NO Controlado q Debemos Controllar y para este caso puntual q es el de validar la Data q Nos llega al Endpoint usaremos   `express-validator`


  - -- Para comenzar con esto
    - Para NO tirar el codigo asi, alegremente, Empezamos x los     `Tests de Aceptacion`    q son los Tests q estamos utilizando para    'Testear CONTROLLERS'
    - Ya con el test pues empezamos a implementar la logica

    - En el Mero INDEX de las ROUTES se coloca el MIDDLEWARE q valida es Schema de Express validator, a ver q no haya errores.
      - `validateReqSchema`
        - Si hay errores lanza el error 422
        - Si NO hay errors continua con el next()
      - Luego se crea el Schema en el Router   `courses.route.ts`   es Schema de validacion de express-validator
        - Ahi lo arma y lo pasa en el   `PUT`   junto con el Validador, ya despues de esto pasa el callback q ejecuta el controller
          - Este controller, recordar q fue INYECTADO y porviene de la app/ mismo
            - Pero este Controller es Quien INYECTA el `Caso de Uso CourseCreator`
              - Este si ya al ser un Caso de Uso, esta en el `Bounded Context` al q corresponde y en el modulo al q pertenece, en este caso:   /Mooc/Courses/application
                - Application xq es 1 caso de uso















## 👤 Modelando el dominio: Agregado Course
### ⛏️ Utilizando objetos Request y Response para comunicarnos con el Application Service
- --- Implementar Objects de Rep y Res para comunicarnos con el App Service
  - -- Como estamos con TDD iniciamos x los tests
    - Asi q modificamos para q reciba 1 Object en lugar de los params x separado
      - Creando asi el   `CreateCourseRequest`  <- /src/context/course/application
        - Esto ya es un  `DTO`    <--  Interface
          - Esto es super util para pasar informacion entre capas
          - Ademas deja listo el Terreno para cuando pasemos a CQRS
          - Estos DTOs NO van a tener comportamiento
            - Asi al tener DTO para Entrada y Salida, ganamos mucho en semantica
              - Asi, como tenemos la API en TS, y tenemos 1 endpoint q hace el trim del video, accion q es mucho mas optima hacerla en Python
                - Pues TS recibira esa Request en el Controller, Armar ese DTO, Tirarlo al al Command Bus y olvidarse de eso. Asincronia pura y dura
                  - El Command Bus va a ser 1 command bus Async q va a ser el responsable de Publicarlo en 1 Message Broker q notificara al proceso de Python q tiene esa task






### 🔮 Refactoring a UUIDs como identificadores
- --- Ahora los ID de cada Course va a ser UUI

  - -- Iniciamos modificando los Tests
    -  Ojo q `Course.ts`  en DOMAIN es el AGREGADO

    -- Creamos el  `Uuid.ts` en el  `shared`
      - Va en  Shared  xq es alto q NO tiene Logica de ninguin Contexto/BC, sino q es una carencia propia de TS ya q no tiene nativo el UUI
        - X eso lo ubicamos en:  /src/Contexts/shared/domain/value-object/Uuid.ts
          - El tener este UUID en el DOMAIN nos da el beneficio de q los Clients sean quein nos pasen ese UUID
            - Ganamos q el CLient/Front W mas facil, ya q desde el incio tendra ese ID
            - Ademas nos facilitamos la vida en los TESTS ya q todo se reducira a comprobacion de igualdad de Valores
              - Como ya mando el UUID en la Request, pues al hacer el test, simplemente compruebo q estos hagan match.

      - Para facilidad d W e integracion con otras librerias, al UUID en DDD con TS lo mantenemos como su valor primitivo q es STRING
        - Llega como 1 String Primitivo, pero debemos validar de q ese primitivo sea un UUID
          - Como TS y Node.js NO tienen una forma nativa de validar y generar UUID, pues recurrimos a librerias externas:    `pnpm add uuid-validate` `pnpm add -d @types/uuid-validate`
          - Modificamos el codigo para q pasen los tests
            - Damos el tipo Uuid en donde corresponda


    -- SUPER OJO
      - `Uuid` esta en DOMAIN de SHARED, pero esta en DOMAIN  <- Arq. Hex.
        - Dada la limitacion de TS al NO tener 1 manera Nativa de crear/validar UUID, lo q nos queda son 2 options
          - Importamos 1 libreria externa q ROMPE x completo con la Arquitectura Hexagonal
            - Pero nos renta mas q crear el algoritmo q lo valide, pero igualmente NO es de DOMAIN e Igualmente se estaria violando en este caso puntual la Arq.Hexagonal
              - Asi q aqui, Dadas las Limitaciones pripias del LP (TS), nos renta mas importar la libreria
          - Creamos nosotros mismos el Algoritmo q lo valide
            - El Algoritmo TAMPOCO es algo de DOMAIN, asi q igualmente estariamos Rompiendo esa parte de la Arq. Hexagonal
              - Seria iluso creer q copiar ese codigo q valida va a evitar q violemos esa ley de dependencias de la Arquitectura Hexagonal

        - X eso, A conciencia sabemos q estamos violando esa Ley de Dependencias de Arq. Hex
          - Y para contener lo maximo posible estos efectos, lo que hacemos es Exponer un Primitivo propio del LP, en este caso   'string'
            - NO estamos exponiendo Nada de la Libreria Externa para evitar terminar Acoplando el codigo a esa dep externa ya ahi si cargarnos todos los principios de la Arq. Hexagonal







### 💌 Constructor de agregados con Parameter Object+Destructuring
- --- Dotamos de mayor semantica con los ValueObjects
  - -- Para > Semantica, para el AGREGADO Course creamos el     `CourseId`   , q extendera de Uuuid
    - Y ademas, al ser 1 Identificador especifico de este Modulo, ira en  `/shared` pero DENTRO del BOUNDED CONTEXT de `MOOC`, xq es propio de este BC.
      - Entonces Crearemos ese `CourseId` en: `src/Contexts/Mooc/Shared/domain/Courses/CourseId.ts`
        - Y este simplemente EXTENDS de Uuui q SI esta en Shared de todos los BC
        - Recordar q solo los Identificadores de Agregados estaran en el Sharen del BC al q pertenencen
          - El resto de ValueObjects mas genericos iran en el Sharen de todo el Context/ como el Uuid.ts

    -- Creamos el    `StringValueObject.ts`   a la altura del ValueObject de Uuid xq es igual de generico y se va a compartir entre todos los BC
      - Este     `StringValueObject`     lo usaran todos los ValueObject q sean String
        - X ejemplo el    `CourseName.ts`
          - Q al ser Especifico del Agregado de Course.ts, pues se coloca en el `Shared del BC de Mooc`, y EXTENDS del este StringValueObject

        -- Este   `CourseName`   es el ValueObject perfecto en donde Colocar las Restriciones de la Logica de Negocio
          - X ej. q el Nombre de `1 curso NO puede tener >30 characters`
            - Entonces, es el VO de   'CourseName'   quien nos debe Asegurar q ese courseName Cumple con esa Regla de Negocio
              - Esto lo `testeamos` como 1 caso Colateral de Testear Nuestros CASOS DE USO
            - Para esto creamos el    `CourseNameLengthExceeded.ts`    q tb es 1 ValueObject a la altura de   'CourseName', y q EXTENDS de:
              - `InvalidArgumentError`   q este si va a la altura del  'StringValueObject' en el Shared general de /Context, y q tb es un ValueObject
                - Este es una EXCEPCION controlada q luego ademas podremos testear con Jest.


    -- `La Unidad de nuestros UnitTest es el Caso de Uso`
      - Asi nos aseguramos de No tener tests fragiles q si cambia esta restricion de Negocio, tengamos q cambiar todo el Test de 1 fichero q solo testee eso
      - Entonces agregariamos 1 nuevo test case
        - Este comprobaria q se Lanza la Excepcion NUESTRA, una Excepcion Controlada x nosotros
          - Y eso lo conseguimos con el    `().toThrow(CourseNameLengthExeeded)`
            - Todo esto el   `CourseCreator.test.ts`   q es el q testea el caso de uso, como ya lo mencionamos antes (la unidad del unit test es el caso de uso)


    -- Aqui podriamos hacer Dependency Injection, SOLID y demas
      - Pero OJO, estamos Hablando del Modelo de DOMINIO
        - Eso querria decir q el q Instancia el CourseId, tiene q recibir la Dependency X Constructor, q a su ves tiene q recibir el Course, y X tanto el CourseRepository en el constructor tiene q recibir 1 UuidValidator, para pasar toda esa cadena de herencia, en los 800 courses q intancies, eso es una Locura, NOO tiene sentido
      - Abogamos x la Composicion


  - Los Value Objects DEBEN estar Dentro de DOMAIN








### 🏗️ Patron ObjectMother para nuestros tests
- --- En este punto tenemso mucho Ruido en nuestros Tests
  - Ademas de todo ese ruido y mas importante, tenemos Fragilidad en nuestros Tests
  - Para solventar todo esto usamos el    `Patron ObjectMother`    + 1 libreria `Faker` para generar valores aleaotrios
    - Al aplicar este Patron ObjectMother, nuestros Tests se Vuelven Mucho Menos Fragiles
      - Xq si Cambia como se hace el New del Course, aqui, en el test ya no tengo q tocar nada, Solo tenego que tocar a quien lo esta Encapsulando que en este caso seria el    `CourseMother`
        - Este Patron de Disenio ObjectMother lo hablamos en el curso de Intro al Testing y buenas Practicas



  - -- Empezamos Creando el `CreateCourseRequestMother` q esta en  '/tests/Contexts/Mooc/Courses/application/CreateCourseRequestMother.ts'  q esta a la altura el propio test  `CourseCreator.test`
    - Va a Generar Request Validas para testear nuestro `Caso de Uso de  CourseCreator`
      - Los Methods aqui son     STATIC   , es decir, Solo le Pertenecen a la Class  `CreateCourseRequestMother`
        - Al ser Static estamos hablando de un   factory method
          - create(): Crea la Req. configurando todos los argumentos como queremos
          - random(): Construye la Req. cuyos valores internos son Aleatorios o Random

        - El    `CourseNameMother`   sera quien valida la logica de negocio para el courseName
          - Empujar las Relgas de Neogocio a la maxima profundidad de nuestro DOMAIN
            - X tanto, este     CourseNameMother    tiene esas validaciones de dominio de q el title debe ser <30 char

      -- Creamos el    `CourseNameMother`  en   '/tests/Contexts/Mooc/Courses/domain/value-object/CourseNameMother.ts'
        - Este   'CourseNameMother'    seria el Patron ObjectMother aplicado al ValueObject CourseName
          - Q sigue la misma estructura del Patron ObjectMother anterior con STATIC Methods
            - Con random y create
              - Esto es convencion de Equipo, si me cambio de equipo, se q me voy a encontrar  random y create
        - Que sera quien genere los courseName en 1ra instancia, respetando esa regla de negocio/dominio, y si es necesario, con 1 method q No lo haga para validad esa causisitica en nuestros tests.
        - Aqui tenemos el     'WordMother'     q es otro ObjectMother q esta especializado en genera palabras aleatorias q ademas parametrizamos para pasarle un MaxLength

      -- Creamos el     `WordMother`    q es nuestro, tenemos el maximo control de eso. Esta en  '/tests/Contexts/Shared/domain/WordMother.ts'
        - Q simplemente genera un Texto Aleatorio q sera el CourseName, aqui estamos acoplados a la libreria Faker
          - Lo q podemos mejorar creando cada method en el   'MotherCreator'
            - Asi c/vez q necesitemos algo de la libreria, 1ro creamos el method en el    MotherCreator     q serviria de FACHADA
              - Asi EVITAMOS q se nos cuele la libreria en nuestros tests
                - Con esto, si cambia la libreia, Nuestros Tests NI se Enteran, y simplemente hacemos cambios en 1 fichero
        - Aqui tenemos otra capa de extraccion, q es el     'MotherCreator'     quien nos extrae de la Libreria
          - Asi descoplamos la dependencia a la libreria
            - Ahora, bien, tenemos q convertir a este    'MotherCreator'    en un  FACHADE
              - Es decir, crear methods propios de este  ObjectMother   q envuelvan a methods de la libreria

      -- Creamos el      `MotherCreator`   en '/tests/Contexts/Shared/domain/MotherCreator.ts'
        - Q debemos convertirlo en un Fachade para NO filtrar la Infrastructure (libreria)

      -- Creamos el resto de   `ObjectMother`   para cada  ValueObject del Domain q nos falta
        - Q serian: CourseIdMother, CourseDurationMother, q son lo q recibe el Caso de Uso
          - `CourseIdMother`        en  '/tests/Contexts/Mooc/Shared/domain/Courses/CourseIdMother.ts'
            - Q seria como en el BC, los Identificadores del Agregado van en el Shared dentro del BC al q pertenecen
              - Q este a su vez necesita del    'UuidMother'    q al igual q en el codigo de produccion
                - Creasmo el    `UuidMother`   en  '/tests/Contexts/Shared/domain/UuidMother.ts'
                  - Va a estar en el  Shared de todos los BC, es decir a la altura de todos los BC
                  - Ademas, como usa 1 method de Faker, pues creamos ese method en MotherCreator para q asi no se filtre la libreria externa (infrastructure) en nuestros tests
                    - Esto es lo q es hacer una Fachade   <--  `MotherCreator.uuid()`
          - `CourseDurationMother`  en '/tests/Contexts/Mooc/Courses/domain/value-object/CourseDurationMother.ts'
            - X ser 1 ValueObject (VO) es de Domain, x tanto replicamos eso como lo hicimos para el   'CourseNameMother'

      -- Creamos el   `CourseMother`    en  'tests/Contexts/Mooc/Courses/domain/CourseMother.ts'
        - Este si ya es el   ObjectMother  del Agregado



  - -- Ahora si podemos modificar el codigo del Test   `CourseCreator.test.ts`
    - Como ya Aplicamos el  `Patron ObjectMother`  ahora nuesto test esta desacoplado
      - Nos desacoplamos de la mayoria del codigo de prod, Solo conoce al  ObjectMother  q necesita
        - El codigo queda mas limpio
        - Los tests ahora son mas Robustos
          - Si cambia aldo, NO se mofician los tests
            - Simplemente vamos a ver q pasa en lso  ObjectMother

    -- Para el Test de q Throw nuestro error controlado `CourseNameLengthExceeded`
      - En este punto Nuestros ObjectMother NO contemplan el test case para este error
        - Xq le estamos pidiendo un CourseName Random con el MaxLength q definamos
        - Asi q lo tenemos q hacer es crear 1 method q lo haga
          - Creamos el Method para NO meter la pata y hardcodear el valor en el Test, ya q eso es pesimo, estariamos ensuciando el codigo y dejaria de ser facilmente mantenible
            - Ya q en todos los tests q requiran validar esta exception, deberiamos hardcodearlo
      - Entonces, en el   `CourseNameMother`
        - Creamos el method    `exceededLength`   q retorna 1 String >30 char para q falle el test
          - NO podemos retornar 1   CourseName   xq ya tengo en el Constructor la regla de validacion de q es < a 30 char
            - Asi, q si yo recibo 1 CourseName, eso ya es valido
      - El   `CreateCourseRequestMother`    es quien hace uso de ese    'exceededLength'
        - Asi q aqui creasmo el  method    `exceededLengthInvalidRequest`
          - Este es quien crea la RequestInvalida para el ExceededLength del cual queremos Testear el   'CourseNameLengthExceeded'    q es nuestra excepcion controlada
            - Propio de la logica de dominio/negocio
        - Aqui al usar  `const request = CreateCourseRequestMother.exceededLengthInvalidRequest();`
          - Estamos usando la Ley de Demeter, xq el nivel de conocimiento desde el nivel 0 (test) es 1.
            - Asi es solo 1 nivel de conocimineto (exceededLengthInvalidRequest)
              - No me voy 1ro al .name().invalidName()
                - Aqui serian 2 saltos, Llamadas Encadenadas
                  - Te Acoplas hasta el Fondo de eso, y esto NOOO puede ser
                    - No puede ser xq nos acoplamos hasta el fondo y eso NO nos da Mantebilidad
              - Ahora, si llegase a cambiar la forma de implementar esa req al course
                - Mis Test NO van a cambiar






    - En este punto, en el `CourseCreator.test.ts` NO estamos acoplados
      - No conocemos Detalles del DOMAIN, simplemente conozco a mi  `ObjectMother` y nada mas
        - Esto es espectacular, queda un codigo super limpio y ademas desacoplado















## 🧐 Guardar en base de datos con Mongo
### 🙊 Integracion de Mongo para guardar en DB por cada Bounded Context
- --- Aqui empiez la Chica
  - Vamos a usar MongoDB para persistir la data
    - Instalamos las deps:
      - No usamos Mongoose, para evitar la penalizacion del Rendimiento
        - Las Deps usan  `^`   q le dice al NPM/PNPM/YARN  q instale todas las updates siempre y cuando No sean Majors
      - Usamo   'convict'   para validar el formato de las EnvV
      - Levantamos Infrastructure a nivel de Docker (MongoDB)
          `pnpm add mongodb convict`

  - Creamos el   docker-compose   para levantar MongoDB




  - -- Creamos el   `index.ts`    para   `convict`  en 'src/Contexts/Mooc/Shared/infrastructure/config/index.ts'
    - Esto se Encuentra en la   INFRASTRUCTURE   compartida x todos los Modules del mismo BC
      - El tener los BC es para replicar lo q tenemos en la empresa y poder asignar departamentos a cada BC sin fricciones, sin q 1 departamento dependa de otro x tener la misma DB
        - `Aqui cada BC tiene su PROPIA DB`
          - Cada 1tiene su INFRASTRUCTURE -> replicar datos x Eventos como veremos en el siguiente curso de DDD en TS
      - Esto esta en el BC (Bounded Context) de  `Mooc`, y aqui estamos leyendo la condiguracion de la DB de 'Mooc'

    - Entonces en est    index    tenemos la URL de conection a Mongo
      - Hacemos el   LoadFile   en base al current path y traemos el JSON del entorno configurado
        - En donde este    moocConfig    tendra esas keys de configuracion q las usaremos para crear nuestros Clientes de DB


  - -- Creamos el   `MongoClientFactory`   q esta en   'src/Contexts/Shared/infrastructure/persistence/mongo/MongoClientFactory.ts' , o en el SHARED general a Todos los BC
    - Este fichero tiene mucho sentido ya q Cada BOUNDED CONTEXT debe tener su Propia DB
      - Esta   Factoria   se dedica a gestionar Todas las Conexiones a DB de Todos los BC, esto referente unicamente a MongoDB
    - El method  `createClient()`
      - Como 1er Argument pide el   BOUNDED CONTEXT
      - El 2do acepta la config del tipo   `MongoConfig`   q contempla 1 URL de Conexion
      - Este method a su vez, Crea 1 POOL de Conexiones para c/BC
        - Aprovecha las conexiones q se abren anteriormente con otras peticiones
        - Le decimos a este factory      'MongoClientFactory'    q me de el Client con este BC o nombre de contexto
          - El `MongoClientFactory.getClient(contextName);`  va a buscar si tiene registrada esa Pool de Conexiones con ese Name, si la tiene te la da, sino 1 undefined
            - Si retorna 1 undefined, pues creamos el cliente y lo conectamos
          - Si viene   undefined   pues Creamos el Client de Mongo
            - Para esto, llamamos a `MongoClientFactory.createAndConnectClient(config);`
              - Q crea la conexion, pero q DEPENDE de Otra Factorya de Configuracion de Mongo
                - La factoria es      `MongoConfigFactory`     q es quien resulve las configs de mongo
                  - A traves del    config    q es el de   CONVICT, le pedimo el  'mongo.url', y eso te devuelve lo que tenga convit
                    - Este    'config'    es el     'index.ts'    q creamos antes
                      - Importamos toda la config de la app, y de toda esa config me quedo unicamente con lo q me interesa q es la mongoConfig q es ese 'mongo.url'









### ✋ Evita que Mongo se filtre en tu dominio
- --- En este punto Ya tenemos Disponible Toda la Canieria de Mongo para aceptar Pool de Conexiones, y siendo un Factory para q si llegase a cambiar algo, todo esta centralizado y todo sea tolerante a cambios y escalable
  - Al final, tenemos el Driver x 1 lado, la conexion, las Factorias
    - Conexiones x BC, etc. Todo lo q vimos en el video anterior

  - -- Procederemos Cambiando el  Test de Aceptacion q tenemos la el Fichero
    - Cambiaremos en el   Inyector de Dependencias
      - En Lugar de Inyectar la de FileRepository, Inyectaremos la de MongoCourseRepository




  - -- Iniciamos desde el Punto mas Externo (Aunq no sea purista TDD, pero es lo q se haria)
    -- Modificamos el    `Inyector de dependencas`   (application.yaml) q tiene el    `Namespace de CourseRepository`     q esta en   '/src/apps/mooc/backend/dependency-injection/Courses/application.yaml'
      - Como tenemos Factorias, debemos REGISTRAR esas FACTORIAS en el Inyector de Dependencias
        - Como son Factorias, se inyectan de otra forma:
          - Empezamos x la Factoria q crea la CONFIGURACION
          - Luego la Factoria q nos CREA las CONNECTIONS
    ```yml
  services:
      # Este namespace NO es 1 Class, sino q vas a usar la Factoria, vas a usar esta Otra clase aparte q es 'MongoConfigFactory' y vas a llamar al method 'createConfig', q lo q te devuelva ese method va a ser la Instancia q el Contenedor del Inyector de Dependencias va a registrar y cada vez q se pida al Inyector de Dependencias con este NamesPace, tu me des esa instancia (resultado del method)
    Mooc.MongoConfig:
      factory:
        class: '../../../../../Contexts/Mooc/Shared/infrastructure/persistence/mongo/MongoConfigFactory'
        method: 'createConfig'

	  # Aqui el method  'createClient'  SI recibe Args, asi q se los pasamos. Igual es un Factory
      # Ags: contextName q en este caso es 'mooc', Config: del inyector de deps de arriba
      # Igual, va a tener en el contenedor del inyector lo q instancie el method en cuestion
    Mooc.ConnectionManager:
      factory:
        class: '../../../../../Contexts/Shared/infrastructure/persistence/mongo/MongoClientFactory'
        method: 'createClient'
      arguments: ['mooc', '@Mooc.MongoConfig']

      # Recibimos x args lo q devuleve el 'createClient', asi q vamos a tener directamente el Cliente de Mongo, NO vamos a porder jugar con las conexiones de otro BC, asi q NO nos podemos saltar las Restricciones q ya hemos definido
      # Aqui Inyecto lo q retorna el  createCliente en el  Mooc.ConnectionManager  namespace
    Mooc.Courses.domain.CourseRepository:
      class: '../../../../../Contexts/Mooc/Courses/infrastructure/persistence/MongoCourseRepository'
      arguments: ['@Mooc.ConnectionManager']
    ```




  - -- Creamos el     `MongoCourseRepository`     en  'src/Contexts/Mooc/Courses/infrastructure/persistence/MongoCourseRepository.ts'
    - Partimos Implementando la Intrface de DOMAIN q es  `CourseRepository`
      - Tenemos el Constructor q como vimos en el Inyector de Dependencias, vamos a Recibir el Client de Mongo


    -- Creamos el    `Nullable`    en 'src/Contexts/Shared/domain/Nullable.ts'
      - Q es 1 Type de TS  q recibe in Generico  <T>


    -- Concepto Aggregate / AggregateRoot
      - El   aggregateRoot   es la puerta de entrada al    Aggregate    q en DDD es 1 `elemento Conceptual`
        - El    Aggregate    NO existe fisicamente, NO tiene 1 fichero
          - Es algo Conceptual de Todas las cosas Relacionadas con Course
            - Dentro de este elemento conceptual SI q me voy a encontrar Ficheros/Classes
              - Ej. El     `Course`    , q es lo q entenderiamos como   `Entity`   de toda la vida
                - Q a su vez tiene sus ValueObjects (VO)
                - Entonces, todo eso, el Course, sus VO Estan Dentro de este Elemento Conceptual q es el Aggregate

          -- Entonces este     Aggregate    es una bolsita llena de cosas a la q le Definimos 1 Puerta unica de entrada, la raiz(root)
            - Esa Raiz sera el Elemento q yo use para Interactuar con cualquier elemento q Esta Dentro del    Aggregate
              - En este caso es la     ABSTRAC CLASS      `AggregateRoot`
                - De la q estariamos extendiendo en todos elementos del   Aggregate   q hacen las de puertas de entrada
                  - Beneficios de esto: Promover    "Tell don't ask"     y promover   "Ley de Demeter"
                    - Es decir, NO hablar con desconocidos, si quiero interactuar con el CourseDuration (q es 1 VO) a quien tengo que Decircelo es a   `Course`   xq es el Unico con el q puedo hablar, ya q es el   AggregateRoot
                      - Veneficios de esto, q si Cambia el Modelado del CourseDuration y ahora pasa a ser 1 Integer, NIGUN CLIENT de afuer de Entera
                        - Solo se Enterara el Agregado Course xq extiende de   AggregateRoot

      -- Creamos el    `AggregateRoot`    en  'src/Contexts/Shared/domain/AggregateRoot.ts'
        - Clase Abstracta q acabamos de explicar, q es la Puerta de Entrada al   Aggregate
          - Tiene el  method    `toPrimitives()`   q nos permitira crear 1 version de datos primitivos de nuestro agregado

        -- Hacemos q     `Course.ts`     EXTENDS   de este   'AggregateRoot'
          - Como es una Abstract Class, se debe cumplir con la firma
            - Asi q implementamos el    toPrimitives()    en esta Entity q viene ahora a ser la Puerta de Entrada al    Aggregate
              - Es decir, si desde afuer quieren saber del  CourseDuration, SOLO a Traves de esta Puerta de Entrada lo podran hacer
          - Este     toPrimitives()    NO es x Mongo
            - Se lo llama asi x mantener ese Nivel de Abstraccion
            - Asi, en el    MongoCourseRepository    le decimos
              - Oye Mongo, lo q tu en tu casas entiendes x    _id    , yo en la mia lo entiendo como ID
                - X tanto NO estamos permitiendo q se Filtren esos Detalles de INFRASTRUCTURE
                  - Esto gracias a este Mapping para evitar q se contamine el DOMAIN


    -- En el    `persist()`     del      `MongoCourseRepository`    usamos el   .updateOne()
      - La ultima opcion es la mas importante    `upsert: true`     ya q nos va a permitir es tener solo 1   .save()   para INSERT/UPDATE.
        - Asi, cuando vamos a Guardar en DB, el   .persist()   va a INSERTAR Siempre y cuando NO exista en DB
        - Si YA Existe en DB, lo va a UPDATE

      -- El method    `search()`    x su parte
        - De la misma forma q antes, accedemos a la Collection de Mongo, pero al tener en 1 Private Method podemos reutilizarlo
          - Ya con la Collection, usamos el    .findOne()   y podemos Tipar lo q nos devuleve este methods para q nos ayude VSCode
        - Ahora q Ya tenemos el    Document   de Mongo, lo q queremos Retornar es el   Agregado (Course)
          - Para esto, usamos x convencion de equipo el method    `fromPrimitives`    q lo Implementamos en el Agregado/Entity     `Course.ts`
            - Ojo q este method  NO  es del    AggregateRoot    , sino q solo es del  Course
            - Simple, hace la busqueda x id en este caso
              - Convencion de equipo: Search Retorna 1 o null
                - SearchMany retorna un array de resultados o un empty arr









### 🏰 Agiliza la creacion de repositorios
- --- Se ha detectado q hay funcionalidades q van a ser comunes para Todos los Repos de Mongo
  - Como podria ser la Persistencia de 1 Agregado en DB, incluso el Search
    - Con lo cual, esto lo podemos Extraer a 1 Clase Base, de la q Van a Extender todos los Repos, para q esos methdos NO los tengamos q reimplementar continuamente
      - Vamos a Compartir Comportamiento x Herencia en vez de x Composicion
        - Esto lo hacemos creando la Clase Base     'MongoRepository'
      - Podriamos hacerlo x Composicion y en vez de Inyectar el Client, inyectar el MongoRepository
        - Pero perderiamo cierta funcionalidad como



  - -- Creamos la Abstract Class    `MongoRepository`     en  'src/Contexts/Shared/infrastructure/persistence/mongo/MongoRepository.ts'
    - Hemos movido los methods comunes de   collection() y persist()  a esta Abstract Class
      - Lo q si hemos cambiado es el     'collectionName()'     q si estaba acoplado a la Collection Concreta del Modulo del BC
        - Asi q ahora dejamos q c/1 defina el name de la collection a la q ataca
    - Este    MongoRepository    es una ABSTRACT Class para EVITAR q se pueda Instanciar
      - Asi solo va a ser utilizada para Herencia
        - Y definimos  Methods  como     PROTECTED    para q los tengan disponibles los Hijos q hereden

    -- Debemos Asegurar q estos methods sean Realmente Genericos y NO esten Acoplados a un tipo de implementacion concreta
      - Esto podemos hacerlo xq los niveles de abstraccion q hemo ido jugando son paralelos
        - Tenemos el     MongoCourseRepository      y tenemos el Agregado      Course
        - Tenemos el     MongoRepository      y tenemos el      AggregateRoot
      - Entonces con quienes van a hablar (`Liskov` Sustitution Principle de SOLID) son esos mismos niveles de abstraccion
        - Entonces, para esto Recibimos 1 Generico <T>  q lo controlamos o limitamos Extendiendolo de   AggregateRoot
          - Asi nos aseguramos d q esa comunicacion sea al mismo nivel de abstraccion
            - Y q Este     `MongoRepository`     tenga los methods del   AggregateRoot
              - Q serian el     `toPrimitives()`



  - -- Ejecutamos los Tests de Aceptacion
    - X eso se crea en DB el course con el name q establecemos en    'Cucumber'
      - Estos tests son de Aceptacion
    - No aparece un name random, xq todos esos random del   faker    los colocamos solo en los   UnitTests
      - X tanto Faltan los    Unit Tests    de la capa de Infrastructure
        - Lo q asumimos o Conocemos como     Tests de Integracion

















## 💼 Bases de datos: Como enfocar los tests y tips para produccion

### ✅ Test de integracion Mongo
- --- Creamos los Tests de Integracion
  - -- Creamos el    `MongoCourseRepository.test`    en   'tests/Contexts/Mooc/Courses/infrastructure/persistence/MongoCourseRepository.test.ts'
  - En este test necesitamos el    'contenedor del inyector de dependencias'
    - Y en base a como funca este inyector, accedemos con el  .get()  y la key/namespace

  - Para mantener nuestros tests lo mas limpios posible creamos 1   'EnvironmentArranger'
    - Q va a ser el q limpie la DB de tests antes y despues de los tests
      - Esto para Evitar el antipatron de tests q son los Tests Encadenados
        - Para HUIR de eso es q necesitamos el     'EnvironmentArranger'



  - -- Creamos el     `EnvironmentArranger`    en   'tests/Contexts/Shared/infrastructure/arranger/EnvironmentArranger.ts'
    - La ubicacion de este test no tiene 1 opinion fuerte xq no replica a la exactitud lo de Prod
      - Pero es la q usa codely
      - No respeta lo de production justamente xq en Prod NO necesitamos estos Arrangerss
    - Este    'EnvironmentArranger'   es `INYECTADO`, asi q lo registramos en el contenedor
      - Lo registramos en el     `application_test.yaml`
        - Aqui creamos el namespace de      'Mooc.EnvironmentArranger'
          - Y le pasamos como Args el Manager de la Conexiones    '@Mooc.ConnectionManager'

    -- La implementacion de este     'EnvironmentArranger'    es una Abstract Class para q NO pueda ser Instanciada, solo sirva para Herencia
      - Tiene 2 abstract methods: Asi tenemos limpios los tests
        - arrange():  inicializar
        - close():    finalizar
      - Lo generalizamos asi xq luego sera reutilizada. Y ademas no es propia de Mongo, sino q tb podra ser usada para PostgreSQL


  - -- Creamos el     `MongoEnvironmentArranger`     q EXTENDS del   'EnvironmentArranger' y esta en    'tests/Contexts/Shared/infrastructure/mongo/MongoEnvironmentArranger.ts'
    - Aqui, como en el Repository vamos a necesitar el   Cliente de Mongo   para poder acceder a DB
      - De la misma manera q en el MongoRepository, Inyectamos el Client x Constructor



  - -- En el     `MongoCourseRepository.test`    ejecutamos el   environmentArranger  tanto en el beforEach como en el afterAll  para limpiar DB y el test sea autocontenido y NO tengamos contamicacion de otros tests
    - El afterAll limpiara de nuevo la DB y Cerrara la Coneccion a DB

    -- Creamos el pimer test case q valida q se este guardando 1 course en DB
      - Aqui es Nomenclatura q en el Test se use  '#'  para hacer referencia al method q estamos testeando. NO es el private de JS
        - Como ya tenemos el     CourseMother     con su methos    random()    podemos crear 1 course asi de facil

      -- Ellos se crearon x la cara el   `MongoClientFactory.test.ts`   'tests/Contexts/Shared/infrastructure/MongoClientFactory.test.ts'
        - Sin explicar como lo hicieron
          - Pero testea q NOO vuelva a abrir Otro Pool de concecciones para el mismo contexto y tal

      -- En el      `controller.steps.ts`   usamos el    'environmentArranger'    para limpiar db y para cerrar coneccion con db xq se abre coneccion y No se cierra

      -- En el    `nodejs.yml`      agregamos la lineas necesarias para levantar el Container de Mongo con Docker
        - Esto para Levantar el Container de Mongo con docker compose
        - Tambien lo bajamos al final
          - GithubActions es tremento, es wow









### 🌼 Tips avanzados para Mongo en produccion
- --- Aprendizajes de usar Mongo en prod
  - Genialy incluso a sido consulto de Mongo

  - -- Bueno, ya Generalizamos el Repository para todo lo q tenga q ver con Mongo
    - Esto con el    `MongoRepository`    q ya lo habiamos implementado
      - El   this.collection()   devulve 1 Collection de Mongo
      - El   persist()   usa el   UPSERT
        - Pero NO lo recomiendan usar en Prod
          - Xq si tiene un Documento tremendo de 100kb y solo cambias el Name, pues le caes a todo el document
            - Es menos eficiente
              - Ademas, es MENOS eficiente ya q en Prod se W con Clusters de Mongo, asi q serain 100kb por Node del Cluster
        - Entonces, aqui el Problema en porduction es mas x el   `{ $set: document },`   ya q NO ataca al attributo en el Caso de Uso de Update
          - Y cae encima a todo el doc
            - Esto el driver de MongoDB NO lo tiene cubierto, pero Mongoose Si q lo tiene
              - X eso genially usa Mongoose en Production

    - Geneally en Prod usa Mongoose xq ya viene preparado contra esa feature de solo modificar el campo/attribute de interes.








### 👀 Optimiza documentos de Mongo de tus agregados
- --- Tip para reducir el size de nuestros docs
  - -- Como vimos en el    `MongoRepository`   seteamos el   'id: undefined'    pero esto Mongo lo interpreta como Null, asi q en el Document SI genera la key id y le setea 1 null
    - Este es el comportamiento x default de mongo
      - Esto es 1 problema aqui en DDD con los Agregados xq recibimos de afuera el UUID
        - Y xq hacemos el   mapping   para NO contaminar nuestro DOMAIN con Infrastructure (_id)


  - --- Para corregir esto, en el    `MongoClientFactory`     en su method   'createAndConnectClient'    hacemos q el client ignore los undefined
    - Con esto, ignora los  undefines  y ya no crea key para ellos, y no lo incorpora al Document
        `const client = new MongoClient(config.url, { ignoreUndefined: true });`
      - Ojo q eso SOLO afecta los   undefined    pero SI Permite nulls

    - En Prod Geneally tiene MongoAtlast q les da metricas q usan para dar seguimiento















## 🐘 Alternativa almacenamiento con PostgreSQL y TypeORM

### ✨ Repositorio para PostgreSQL con TypeORM Abstrayendo ValueObjects
- --- A menos q tengas 1 razon de peso para dejar SQL e ir a NO SQL, una DB Relacional te basta y sobra
  - -- TypeROM, en este caso SI q queremos beneficianos de las features q ya nos da 1 ORM
    - Instalamos deps:  `pnpm add typeorm`
    - Seguiremos los mismo niveles de indireccion y abstraccion q vismo con Mongo
      - Es decir tener los Repository y los Genericos



- -- Creamos el     `TypeOrmCourseRepository`   en 'src/Contexts/Mooc/Courses/infrastructure/persistence/TypeOrmCourseRepository.ts'
  - Q es la Abstract Class para todos los Repository q implementen TypeORM tengan cierta funcionalidad compartida
  - Tenemos algunas diferencias con respoecto al de Mongo
    - Aqui como ya usamos 1 ORM, Pedimos el Schema q es lo q utilizamos para q TypeORM sepa como hacer el maping de las tablas a nuestros Agregados
    - Aqui se evidencia la abstraccion robusta q hemos hecho a la   Interface    ya q Collection NO es parte de ella, xq aqui en TypeORM xq ejemplo, la collection NO tiene sentido

  - Tenemos los sig methods:
    - Con    `repository()`    obtenemos el Repository correspondiente al Schema particular q requiramos
      - Esto ya Nos return el Repository de TypeORM correcto y tipado gracias al generico  <T>
    - El    `entitySchema()`    retorna el Schema de la Entity q necesitamso. En este Caso de Course

  - Implementamos el Patron de Disenio Template Mehtod
    - Para q TypeORM sepa a q Schema estamos atacando
      - El    `entitySchema()`    retorna el Schema de la Entity q necesitamso




  - -- Creamos el      `TypeOrmRepository`     en  'src/Contexts/Shared/infrastructure/persistence/typeorm/TypeOrmRepository.ts'
    - Al igual q con Mongo,  INYECTAMOS  el Client de TypeROM


  - -- Creamos el      `CourseEntity`   en   'src/Contexts/Mooc/Courses/infrastructure/persistence/typeorm/CourseEntity.ts'
    - Q es la Entity de toda la vida q hace el mapping el Agregado a Tablas
      - El Mapping d los 1 mundos, las Properties de mi Agregado y las Columnas de mi DB
      - Para c/column tenemos 1   `transform`   q en TypeORM exigen q sea 1 Object q tenga 2 methods
        - to y from
        - Como en este caso del    Agregado Course    los 3 campos con   ValueObjects   hay algo en comun q tiene todos ellos a la hora de persistir y leer
          - Q es q Todos estos VO tiene el campo   value   q es el primitivo


  - -- Creamos el     `ValueObjectTransformer`    en  'src/Contexts/Shared/infrastructure/persistence/typeorm/ValueObjectTransformer.ts'
    - Como TyORM nos pide q tengamos 1 Obj (x eso el return {}) q tenga   to y from
      - `to`: El valor q queremos persistir, lo convertimos a Primitive
        - Como nos llega el VO, el Primitivo lo tenemos en el    .value
      - `from`: Nos da el Primitive de DB, y lo transformamos en ValueObject xq son VO en este caso
        - Obtenemos el Primitivo de DB q creamos el VO con el new
          - Nos aseguramos q el  new  sea publico (Clase Instanciable)  con el     'NewableClass'
    - En los params recibe 1    ValueObject    q es del tipo      'NewableClass'
      -- Creamos este      `NewableClass`      q es 1 interface en   'src/Contexts/Shared/domain/NewableClass.ts'
        - Aqui NO estamos pasndo la Instancia en Concreo del VO xq lo q queremos es Crear la Instancia en Automatico, sin saber la Implementacion en concreto
          - Para conseguir eso, subimos el nivel de abstraccion a     'NewableClass'
            - Esta clase q le pasemos Debe ser Instanciable, debe tener el   new   publico
              - Aun asi necesitamos saber q tipo de Class es, x esto el    'NewableClass'    recibe 1 Generico   <T>   q lo acotamos a    Fucntion   , y retornamos ese mismo Generic




    -- Creamos el     `ValueObject`    en  'src/Contexts/Shared/domain/value-object/ValueObject.ts'
      - Tiene sus methods y demas
        - El   'ensureValueIsDefined'    NO contempla valores de   null o undefined

      - Es la Capa mas alta del nivel de Jerarquia para los VO
        - De este EXTENDS     `StringValueObject`   y asi ya se cumple el Type de    'ValueObjectTransformer'     en     'CourseEntity'
          - Entonces modificamos su implementacion dado q la funcionalidad de toString() ahora la delegamos al    ValueObject.ts

      - Asi mismo EXTENDS     `Uuid`     en   'src/Contexts/Shared/domain/value-object/Uuid.ts'
        - Igualmente modificamos el codigo y queda mas limpio ya q mucha de la funcionalidad la delegamos al parent en esta jerarquia de VO q es el      'ValueObject.ts'
        - Cabe mencionar q modificamos este uuid tb xq de este  Extends    "CourseId"     y necesitmoas q este VO tb cumpla con el  type  q tide la   Entity   en el     "ValueObjectTransformer"





  - -- Pool de Conexiones a DB
    -- Creamos el     `TypeOrmClientFactory`    en  'src/Contexts/Shared/infrastructure/persistence/typeorm/TypeOrmConfig.ts'
      -


    -- Creamos el     `TypeOrmConfig`    en  'src/Contexts/Shared/infrastructure/persistence/typeorm/TypeOrmConfig.ts'


    -- Creamso el   `TypeOrmConfigFactory`   en  ''
      - Q es la Factoria de Configuracion
        - Donde la config sigue siendo la de   Convict
      - Requiere la config en el   `index.ts`   de  /config   en  'src/Contexts/Mooc/Shared/infrastructure/config/index.ts'
        - Establecemos toda la Config



  - -- TypeORM nos ha obligado a cambiar la
    - TypeORM nos obliga a q el   Constructor   del Agregado reciba los Args x separado
      - X eso cambiamos este codigo
        - OJO q no siempre debemos cmabiar nuestro dominio x 1 libreria externa
          - En este caso lo hacemos xq desde el inicio dijimos q No tiene Sentido agruparlos en un Obj, asi q no supone mayor cambio
            - Esto NOOOOO hace para otros casos con otras librerias xq Destruyen nuestro dominio y se va a tomar x saco DDD


  - -- Nos faltaria hacer ajustes en el Inyector de Dependencias









### 🤟 Test de integracion PostgreSQL
- --- Iniciamos con los Tests de Integracion para PostgreSQL
  - La Estructura de test de es muy similar a la de Mongo
    - Pero con sus diferencias

  - -- Debemos configurar el     Inyector de Dependencias     para q nos retorne el de TypeORM
    - Para q inyecte todas las factorias y deas de TypeORM


  - -- Creamos el     `CourseRepository.test.ts`     q testeara como el q hicimos para mongo
    - Pero aqui se le Inyectara el    TypeORMFactory y demas
    - Tb creamos su propio     arranger     especifico para TypeORM


    -- Creamos el     `TypeOrmEnvironmentArranger`   en  'tests/Contexts/Shared/infrastructure/typeorm/TypeOrmEnvironmentArranger.ts'
      - Se le Inyecta el Client x constructor al igual q con mongo

    -- Creamos el      `TypeOrmClientFactory.test.ts`   en   'tests/Contexts/Shared/infrastructure/TypeOrmClientFactory.test.ts'















## 🔜 Conclusion y siguientes pasos

### 🤯 Los tests me engañan
- ---
  - -- Los    `ObjectMother`    Instancias Entidades de DOMAIN SINCRONAS


  -- Geneally No ha tenido buena experiencia con los Mocks
    - Esto se podria solventar con    CQRS    para recibir el evento y solo replicar data q nos interesa
      - Sin embargo no simpre va a ser de facil, xq existe Casos de Uso q x naturaleza son complejos

    - En ciertos casos muy complejos, y Caso de Uso debera hacer uso de varios repository
      - Pero lo q aqui se enfoca en DDD y Arq. Hex es q 1 caso de uso solo tenga 1 Repo
        - NO OneToMany ni cosas asi en estos enfoques arquitectonicos




### 💪 Conclusiones y siguientes pasos
- --- Recomienda otros cursos
  -- Ahora vendria el de DDD en TS con CQRS

  - Recomienda verlos de DDD con Java y PHP aunq no W con ellos para nutrirnos de conceptos de otros ecosistemas

  - Recomienda Todos los cuross de Refactoring
















## Doc de mi Repo de Github
- --- MAIN
  - Tiene todo Ok para TypeORM, pero Borre TODO lo de MONGO
    - Ademas tiene una version nueva de TypeORM, PERO todo lo q uso esta Deprecated todo lo q se usa en los Factory y demas   'TypeOrmClientFactory'
      - Sin los deprecated todo va bien, pero el test de las connection del Pool de conexiones da error
        - Q es el mas importante para rendimiento
          - Q use 1 conexion abierta del pool, y no cree y cirre conexiones x cada request
    - Todos sus tests y demas xq generaban conflictos ya q se inyecta el Factory de TypeORM y no el de Mongo
      - Asi q daba error sus tests
  - Ultimo Commit / last commit:
    ```git
      * d07e53e (HEAD -> main, origin/main) [ADD]: Integration tests TypeORM (without mongo tests or files)
    ```


  -- YA RECUPERO los archivos de Mongo
    - Con razon dijeron q es tan facil como cambiar la clase y el inyector de dependencias
    - Se cambia eso y todo corre perfecto
      - Xq ahora en los Tests perisiten en DB usando el Inyector de Dependencias q apunta a TypeORM
      - X esto es GLORIA VENDITA teners tests tan Robustos con ObjectMother y demas
        - En este punto      `MongoCourseRepository.test.ts`     es exactamente lo mismo q      `CourseRepository.test.ts`
          - Los 2 usan el  ObjectMother
          - En este punto el de Mongo Solo tiene el Nombre xq NOOOO usa mongo ya q el Inyector de dependencias apunta a la factoria de TypeORM, asi q hacen lo mismo practicamente









- --- typeorm-ok-old-version
  - Tiene todo OK con 1 OLD VERSION de TypeORM  (v < v0.3.x)
    - Aqui si podemos usar los connections y demas

  - last commit:
      * f185be8 (typeorm-ok-old-version) pasa todo, old version de typeorm con connections y demas




- --- 2-typeorm
  - Todos los tests FALLAN xq DEBO ACTUALIZAR el inyector de dependencias
  - Pero es la MADRE de todos los files, asi q es el mas completo en Files con Mongo y TypeORM
  - El testo de ramas son implmentaciones para type orm q eliminan files, pero el MAIN ya lo arregle, me falta el old-version
    - Pero este NO es solo copiar y pegar los tests, xq reescribiria los cambios q hacen q pasen los tests para esa version de TyORM












#### Repo del curso
-- Cada seccion tiene su carpetita y cada clase de la seccion tambien
  - Con lo cual es facil de seguir

      https://github.com/CodelyTV/typescript-ddd-course



  -- Repo d 1 student q usa otro inyector de dependencias
    - tsyringe
        https://github.com/devlegacy/ts-clean-architecture


