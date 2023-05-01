# DDD en TypeScript: Comunicacion entre servicios y aplicaciones



## 🛰️ Comunica aplicaciones y servicios con TypeScript y DDD

### ✉️ Comunicacion con eventos de dominio en aplicaciones DDD
- --- Inciamos con el Codigo final del curso anterior
  - Aqui tenemos el Caso de Uso   Crear Cruso    (CourseCreator)
  - Como ahora necesitams 1 counter, lo q normalmente se haria, es modificar el Repository para hacer ese  select count(*)   en DB

  - Pero aqui vamos a implementar algo mucho mas escalable en donde Separamos este   CorusesCounter  en un nuevo Modulo
    - X ello, necesitamos Comunicacion entre Modulos/Servicios
      - Con lo cual debemos Modificar este     Caso de Uso    `CourseCreater`    para q en lugar de Solo guardar el Course, tb debera INFORMAR/NOTIFICAR o    `PUBLICAR 1 Evneto de Dominio`
        - Que seria el de   crear 1 curso      `course_created`















## Comunicacion entre modulos con EventBus asíncrono
### 👇 Definicion de eventos de dominio y subscribers
- --- Como queremos Publicar Eventos de Dominio y poder Suscribirnos a ese evento, pues al hablar de Publicar Eventos y demas, de inmediato lo relacionamos con RabbitMQ, Kafka, etc (brokers)
  - Pero como estamos en    DDD    nuestro foco es el    DOMAIN    vamos a intentar modelar los Elementos q va a W nuestro Domain para ver como se va a manejar la complejidad
    - Modelar los elementos q vamos a necesitar para esa Comunicacion  Event-Subscriber




- --- Para este tipo de modelado se nos agrega Complejidad Extra ligada al propio LP TS
  - -- Comenzamos con la creacion del       `DomainEvent`      q es una Abstract Class para q NO se pueda instanciar ya q no tiene sentido 1 instancia de esta clase sin su especializacion. Esto lo creamso en   'src/Contexts/Shared/domain/DomainEvent.ts'
    - Esta AbstractClass es muy general ya q de ella Extenderan las clases especializadas
    - Contiene la informacion q las clases q hereden de ella van a compartir como el id de agregado, el eventid, fecha y nombre del evento
    - Esta en Shared, pero dentro de Domain, es 1 elemento de dominio puesto q es de la Logica de Negocio
      - Como estos eventos van a viajar, debemos implementar la forma en como estos events se Serializan y Deserializan, esto con el      frontPrimitives()    y     toPrimitives()
        - `toPrimitives()`: Retornara las Propiedades Especificas de Cada Evento
          - No va a Return de las Propiedades del    'DomainEvent'    sino q la propiedades especificas del Evento
            - Return de los datos concretos q han producido ese evento
              - Si creo 1 curso, retorna los datos/properties del Course
              - Datos especificos
        - `fromPrimitives()`: Llega 1 mensaje q esta serializado y trae las propiedades genericas, propias de este   'DomainEvent'   , ademas del   `attribute: DomainEventAttributes`, q son las porperties especificas del Evento
          - Este DomainEventAttributes es un ANY xq no encontraron una forma d q validasen los tipos

    -- Entonces, tenemos 1 Evento de DOminio, q para enviarlo x 1 sistema de mensajes tengo q Serializarlo y para ello uso el     'toPrimitives()'
      - Si me      Suscribo     a 1 Evento, lo q me viene del sistema de mensajeria es algo Serializado, asi que debo   Deserializarlo   con el      'fromPrimitives()'    para poder tener el Obj Domain Event y ya poder W con eso

    -- Tambien Exportamos el Type     `DomainEventClass`
      - Representa las Propiedades Estaticas de la clase    "DomainEvent"   para sobrescribir estas propiedades estaticas x cada Evento de Dominio
        - Esto nos ayuda mas q nada en el Tipado para poder Sobrescribir correctamente



  - -- Creamos los Subscribers, comenzando x    `DomainEventSubscriber`    en   'src/Contexts/Shared/domain/DomainEventSubscriber.ts'
    -- Tenemos methods simples:
      - `subscribedTo()`: Me informa en q eventos estoy interesado como Subscriber
        - Este method Return de 1 Arr de   `DomainEventClass`   q es el type q esamos exportando en   'DomainEvent'
          - Esto para tener el tipado de su las propiedades estaticas
      - `on()`: Se va a ejecutar cuando llega 1 Evento de Interes



  - -- Creamos el event bus     `EventBus`   en  'src/Contexts/Shared/domain/EventBus.ts'
    - Q es el Encargado de  Publicar los Eventos
      - X tanto, en principio no necesitaria mas q el    `publish()`   para publicar eventos

      -- Pero aqui tenemos 1 Particularidad x la q necesitamos el      `addSubscribers()`
        - La necesitamos xq el Contenedor de Inyeccion de Dependencias q estamos usando tiene la Limitacion de  NOO  manejar Dependencias Circulares
          - Imagina q tu das de alta 1 Subscriber q utiliza 1 Caso de Uso q a su vez utiliza el EventBus
            - Ahi tienes una dependencia circular q   'node-dependency-injection'  no puede manejar
            - Como el Inyector de Dependencias No lo gestiona bien, tenemos q romper esa dependencia circular
              - X tanto, en lugar de pasarle los    Subscribers x el Constructor   , 1ro Instanciamos el Event Bus, y despues le pasamos los Subscribers
                // TODO: Ver como corregir esto con  'tsyringe'  o  'awilix'
        - Entonces, en 1ra instancia Abogariamso x Inyectar x Constructor, pero x el inyector no podemos
          - Event bus sigue el Patron Mediator



  - -- Creamos la Implementacion de Event Bus, q iniciamos con 1 en Memoria    `InMemoryAsyncEventBus`   en   'src/Contexts/Shared/infrastructure/EventBus/InMemory/InMemoryAsyncEventBus.ts'
    -- Al final Implementar 1   `Event Bus`   es bastante sencillo
      - Lo q tiene q hacer es, cuando me llega 1 Event ejecuto 1 Subscriber
        - Al final es 1 Mapa de Relaciones
      - Aqui, la ventaja de estar en   Node.js  es q podemos usar la      Native Asynchrony      q ya tiene, x tanto hacemos uso de     `EventEmitter`   propio de node.js
        - Nos ayuda a: Llega 1 Evento, Ejecuto el q Esta Interado en ese Event

    -- Extender de este    `EventEmitter`   nos da acceso a los methods:
      - `emit()`: q ejecuta la accion de Emitir el Evento
      - `on()`: crea la Relacion entre 1 Event y 1 Ejecutor

    -- Implementamos la Interface de   'EventBus'
      - `publish()`: X c/1 de los Eventos q nos llegan llamamos al   "emit()"
      - `addSubscribers()`: x c/Subscriber q nos llega y c/Events en lo q Esta Interesado ese Subscriber ejecutamos el    "on()"   para q cree la relacion

      -- En el   `addSubscribers()`    tenemos acceso al    `event.EVENT_NAME`   gracias al   `DomainEventClass`
        `this.on(event.EVENT_NAME, subscriber.on.bind(subscriber));`
          - Aqui es donde cobra sentido el haber creado el     'DomainEventClass'    ya q si en el     "DomainEventSubscriber"    en lugar de retornar el     'DomainEventClass'     return del     'DomainEvent'    x TS no lo entiende correctamente
            - Es decir, dado q el     'DomainEvent'    es 1 Abstract Class, TS No tiene acceso a las properties ya q lo TS en este momento NO Soporta q las Instancias de Clase tengan acceso a las Propiedades Estaticas de la Clase
              - X eso es q    `DomainEventClass`   nos TIPA la Class y NO 1 instancia de la Clase
                - Esto xq el    subscribe()   del   DomainEventClass  retorna las CLASSES de los Eventos a los q esta interesado, y NOO las intancias
                - Asi tenemos seguridad en el Tipado








### 🥳 Publicacion de eventos de dominio CourseCreated
- --- En este punto ya tenemos implementada toda la logica del EventBus en memoria aprovechando de la Asynchrony Nativa de Node.js
  - En este punto ya podemos crear el Caso de Uso     CorusesCounter     q tendria su propio Modulo


  - -- Vamos a empezar consiguiendo q Nuestro      Aggregate  `AggregateRoot`      almacene los Eventos de Dominio q suceden relacionados con el agregado y para esto debemos MODIFICAR el    'AggregateRoot'
    - Para agregarle   1 methods    `record()`    q lo q va a hacer es q Cada vez q Llamen al Agregado/Entity con 1 Evento, lo registra y almacena
    - `record()`: Registr y almacena el Evento
    - `pullDomainEvents`: Retorna los Eventos q han sucedido y los Elimina de su registro

    -- Como estamos modificando directamente el     Aggregate    , modificando el   `AggregateRoot`
      - Le estamos danto a todos nuestros      Agregados/Entities     la capacidad de registrar Eventos
      - Con lo cual, en este punto ya podemos implementar eso a nuestro Agregado/Entity    Course


  - -- Modificamos el Agregado/Entity     `Course`
    - Con esta modigicacion al   Aggregate   ahora tenemos 2 methods factoria:
      - Se agrega 1      Metodo Factoria     `create()`
        - Este metodo factoria es el q se va a llamar cuando se Crea 1 Curso nuevo
        - Esta creando 1 Course q Antes NO exisitia
      - fromPrimitives():
        - Q instancia 1 course q YA Existia pero q estaba almacenado

    -- `create()`
      - Como es el q Esta Creando desde Cero el Course, aqui Si Puedo Registrar el Evento de q estoy creando 1 course xq solo puede sucrder aqui
        - SOLO aqui se puede Registrar el Evento de Crear el Course
          - X eso aqui usamos el    `record()`   para registrar el Evento

      --- Con esto, dado q estamos en   DDD   tenemos  la Creacion de los "Eventos de Dominio" DENTRO de la  Entity/Agregado
        - Asi es muy facil entender la logica, y saber en donde se estan creando los eventos
        - El     pullDomainEvents    NO lo implementamos aqui, sino q en el     "CourseCretero"
          - A la hora de Publicar el evento
            - Cuando PUBLICAR este Evento?  Para eso tenemos el Caso de Uso     `CourseCreator`


  - -- Creamos el     `CourseCreatedDomainEvent`    en   'src/Contexts/Mooc/Courses/domain/CourseCreatedDomainEvent.ts'
    - Q Herda/Extends de    'DomainEvent'    y q gracias a como esta implementado, pasa sus properties
      - Porterties propias de este Evento como el     duration, name    y el Id del course es el Id del agregado xq course es 1 agregado
      -



  - -- Como estamos siguiendo TDD, antes de modificar el     Caso de Uso CourseCreator, vamos a modificar los tests
    -- Modificamos el test     `CourseCreator.test.ts`
      - Aqui vamos a tener 1 nueva dependencia q es el    EventBus
        - Cuando tenga una Nueva Dependencia de la q necesite informacion, se genera 1 Moc
          - Y ahi es donde encapsulo las preguntas q le quiero hacer

    -- Creamos el     `EventBusMock`    en  'tests/Contexts/Mooc/Courses/__mocks__/EventBusMock.ts' e implementa la Interface  EventBus
      - El method    `assertLastPublishedEventIs()`    recibe 1 Evento y me va a hacer el    assert   d q el Evento ha sido Publicado

    -- De esta forma, nuestro test    `CourseCreator.test.ts`     queda tan sencillo como crearle 1 una linea q invoque el assertLastPublishedEventIs
      - Con lo cual NO cambiamos Ni la preparacion Ni la ejecucion del test, sin q unicamente cambiamos la assercion
      - Con esto solo se le mete 1 nuevo     Colaborador    q es el    EventBus
        - Y la nueva asercion d q el EventBus debe Publicar 1 domainEvent
          - Este    domainEvent    lo creamos con el    ObjectMother    `CourseCreatedDomainEventMother`
            - Creamos este ObjectMother para q los tests sean mas robustos y todo lo q vimos en el curso anterior de DDD con TS

    -- Creamos el ObjectMother    `CourseCreatedDomainEventMother`    en 'tests/Contexts/Mooc/Courses/domain/CourseCreatedDomainEventMother.ts'
      - Q x Convencion, al igual q el    "CourseMother"    tiene los methods:
        - create()
        - fromCourse()



  - --- Ahora q ya tenemos el Test, podemos ir al     `Caso de Uso CourseCreator`    y modificarlo
    - -- Aqui SI q PUBLICAMOS los Eventos de Dominio
      - En el Agregado/Entity creamos o REGISTRAMOS los Eventos
      - En los Casos de uso PUBLICAMOS estos eventos registrados x el Agregado/Entity

    -- Entonces, tenemos la Nueva Dependencia del   `EventBus`    x constructor y en el
      - `run()` Agregamos la nueva linea con la q PUBLICAMOS Solo los Eventos q el Agreago/Entity nos ofrece
        - Aqui NO nos intersa saber Q Eventos se estan Produciento/Registrando xq eso es Responsabilidad del Agreago/Entity

      - Con el test, Simplemente acabo de Ejecutar 1 Caso de Uso, el Agregado/Entity Gestiona Q eventos necesita Emitir/Registrar dado ese Caso, y este Caso de Uso los PUBLICA
        - Aqui ya tendriamos la Publicacion del Evento
          - Este method   `course.pullDomainEvents()`   es el tipico method q viola la regla del cqs (viola entre comillas)
            - La regla de q O bien eres 1 comando o bien eres una query
            - En este caso, este method Obtiene datos xq los recibe y Muta el estado
              - Esto Tiene Sentido xq los Eventos los queremos 1 unica vez
                - Con lo cual, si vuelvo a llamar este method en la siguiente linea, ya NO habran eventos q gestionar xq recuerda:
                  - El   "pullDomainEvents"   del    "AggregateRoot"  tb LIMPIA los eventos



  - -- Aqui la Desicion es Importante
    - El      Agregado/Entity      es quien    Emite/Registra los Eventos
    - El     Caso de Uso       es quien     PUBLICA los Eventos








### 🤔 Subscripcion de eventos y test de aceptacion
- --- En este punto YA tenemos Nuestro Evento   PUBLICADO   en el EventBus, ahora vamos a SUSCRIBIRNOS a el, e Incrementar ese contador de Courses (CorusesCounter)
  - s



  - -- Empezamos x el    `Test de Aceptacion`    q es con Lenguaje Gheerkings y Cucumber
    - Creamos el     `get-courses-counter.feature`    en  'tests/apps/mooc/backend/features/courses_counter/get-courses-counter.feature'
    - Para esto tenemos una API q nos permite recuperar el valor del CorusesCounter
      - En nuestro Test de Aceptacion tenemos:
        - Dado 1 EVENTO de     course_created     queremos q nuestro    CourseCounter    nos retorne un valor de 1
    - Aqui pasamos el message en JSON
      - Q perfectamente se entiente como el cuerpo de 1 req http a 1 controller
    - Como es 1 Nuevo Test de Aceptacion, debemos implementar los    `steps`   de CUCUMBER para Enviar Eventos


    -- Creamos el      `eventBus.steps.ts`       en 'tests/apps/mooc/backend/features/step_definitions/eventBus.steps.ts'
      - Recuperamos el   EventBus   del Mook Backend a traves del Inyector de dependencias
      - El 1er    Given    del step va a tomar el JSON q le envia la feature
        - Lo va a    deserializar   y lo va a    PUBLICAR    en el   EventBus

      -- El    `.deserialize()`     es necesario xq mas adelante meteremos RabbitMQ
        - Lo q va a hacer es, Dado 1 Tipo de Evento, va a buscar la CLASS con la q esta Relacionada y va a llamar al    fromPrimitives()   de esa Class
        - Como estamos usando 1     EventBus Async    , vamos a tener q esperar
        - Si no fuera asi, nos basta con el     fromPrimitives()    q ya tenemos


      -- Entonces, Creamos el      `DomainEventDeserializer`     en 'src/Contexts/Shared/infrastructure/EventBus/DomainEventDeserializer.ts'
        - Este elemento va a ser necesario a futuro, cuando Implementemos un     EventBus    q vaya x Infrastructure
        - En este momento no necesito serializar y deserialziar xq estoy con   EventBus   en memoria ya q directamente voy W con las Instancias q tengo
          - Pero esto es Necesari para cuando apliquemos RabbitMQ, y ademas, en este momento SI q es Necesaria
            - Puesto q el     Test de Aceptacion    envia 1 JSON, asi q para ese test SI q necesito serializar y deserializar
              - Y ya de paso me queda para futuras clases

        - El   `deserialize`   va a Parsear/Parse el JSON, extraer sus  Attributes y Recuperar la Class q necesita
          - Tiene 1    `super.get()`     xq esta Class    EXTENDS   de   MAP
          - Entonces, cuando la creo, la voy a configurar con el   `configure`
            - Coge todos los Subscribers q tenga en la app
              - Nos Basamos SOLO en los   SUBSCRIBERS   xq NO necesito Deserializar todos los Eventos q hay en la App
                - SOLO los q voy a Escuchar


      -- Creamos el       `DomainEventSubscribers`      en 'src/Contexts/Shared/infrastructure/EventBus/DomainEventSubscribers.ts'
        - Q esta ligado al   'DomainEventDeserializer'    ya q asi nos traemos SOLO los Eventos q voy a Escuchar
        - Encapsula la Logica de:
          - Coge Todos los Subscribers en este Tag Container
            - Les aplicamos 1 Tag para NO conocerlos todos
          - Los recuperamos, luego, x c/1 cogemos la Instancia del Inyector de Dependencias
            - Y la guardamos en la propiedad    items
        - Este   TAG    va en el    ymal    del inyector de dependencias   q esta en   'src/apps/mooc/backend/dependency-injection/`CoursesCounter/application.yaml`'
          - Aqui simplemente agregamos el tema de los    TAGS     , puesto q tenemos el Subscriber q recibe el Caso de Uso  x arguments
            - Tenemo el    TAG    q es muy util a la Hora de Agregar Nuevos Subscribers
              - No vamos a tener q tonar nada en la forma en la q se registran
                - Ya q SOLO x Tener este Tag, vamos a estar registrandolo en el   EventBus



    - -- Ahora nos falta el   Subscriber
      - Lo construimos con todo el Module     `CoursesCounter`      'src/Contexts/Mooc/CoursesCounter'




- --- A partir de Aqui Uso el Repo de Ellos xq NO me funcino el tests, me da errores con q no es iterable no se q cosa del EvenBus
  - Probe copiando y pegando src y tests, pero me da el error de Tipos en el   MoocBackend
    - Esto con el EventBus y demas










### 🚷 Idempotencia a la hora de consumir eventos
- --- Tenemos el Sistema Funcionando
  - Tenemos el     Course     module PUBLICANDO el Evento
  - Tenemos el     CoursesCounter       module Escuchando y Reaccionando al Evento
  - Asi ya tenemos nuestro Sistema Escalable
    - Con Tablas independientes


  - -- Si este sistema de    EventBus    q implementamos en Memoria lo llevamos a Sistemas de Mensajeria como   RabbitMQ
    - U otros sistemas de Mensajeria q x dentro funcionan con sistemas distribuidos
      - NO nos pueden garantizar q el Mensaje SOLO se entrege 1 vez
        - Sino q nos puede llegar varias veces el mensaje a nivel de Subscriptor
    - Audince al usa AWS a una escala grande, este NO garantiza q los Mensajes Solo lleguen 1 vez, ni q lleguen en orden


  - -- Empezamos con los Tests de Aceptacion
    - A estas alturas nuestro sistema nos permite ejecutar este tipo de tests


    -- Modificamos el     `get-courses-counter.feature`     agregando el test de q se Envia el Mismo Evento +1 vez.
      - En este caso, creamos 2 cursos xq estamos enviando 2      `aggregateId`     distintos
        - Pero 1 de ellos 2 veces, asi q debemos hacer q el Container solo se incremente 2 veces
          - Ya q SOLO son 2 crusos los q se van a crear
            - Esto valida q tengamos controlada esta causistica de q se envie +1 vez el Mismo Evento

      -- Si No modificamos la logica del     CoursesCounterIncrementer     el test dice q se crean 4 cursos en lugar de 2
        - Esto xq NO estamos controlando el problema de mensajes repetidos
        - Esto xq siempre q se ejecute este Caso de Uso, el contador se va a incrementar
          - Para corregir esto, metemos 1 ifazo :v  (if)


      -- Modificamos el     `CoursesCounterIncrementer`
        - Como el      CoursesCounterIncrementer   tiene los Courses q esta teniendo en cuenta para el Conteo
          - Vamos a preguntarle al   coursecounter, oye, ya has incrementado este Course? ya lo has tenido en cuenta?
            - Si NO lo has tenido en cuenta, entonces Incrementalo
        - De esta manera Controlamos los eventos repetidos con la misma info
          - Asi evitamos q los Mensajes duplicados nos generen problema de consistencia en nuestros datos
          - Esto gracias al   `.hasIncremented()`    q hace uso del    `existingCourses`  q es el Array de Ids q tenemso a disposicion
        - Al hacerlo asi NO tenemos q montar infraestrcutura extra para saver si se repite el evento con la misma info
          - Xq si se monta infrastructura seria con 1 redis, q consulte en base al id del evento y demas
          - Asi controlamos este tema de Idempotencia localmente, sin necesitad de infrastructura adicional

        -- En Audinse lo hacen asi, xq consideran que es una Regla de Negocio q NO se puede ejecutar +1 la misma accion sobre el Mismo Agregado/Entity
          - Asi pues las reglas de negocio tiene q estar en el dominio

        - Se debe entender q la Idempotance se debe manejar segun el caso de uso
          - Si es Creacion como en este caso, la manejamos asi, con el   ifazo
          - Si es Edicion
            - Deberiamos colocar la Fecha del Ultimo Evento q hizo esa modificacion
              - X si nos vienen desordenados, haber si se tiene q descartar
              - Debemos pensar asi


    -- HAsta este punto, x el     EventEmitter     de Node.js no teniamos problemas con esto
      - Pero esto SI q sera 1 problema cuando metamos   RabbitMQ















## 🐇 Comunicacion entre Bounded Context con RabbitMQ
### 👽 Publicar eventos de dominio en RabbitMQ
- --- Pese a q en este punto nuestro Dominio No es muy complejo, preveemos su crecimiento y habilitamos 1 forma de comunicarse con otros departamentos, otros servicios y demas.
  - Para lograr esto, debemos implementar 1 EventBus q salga de la ejecucion en memoria y se vaya x una Infrastructure externa
    - En este caso RabbitMQ.
    - Esto ademas, ayuda al Balanceo de Carga, q si tenemos en memoria NO lo podriamos hacer



- --- Creamos el     `RabbitMQEventBus`      en 'src/Contexts/Shared/infrastructure/EventBus/RabbitMQ/RabbitMQEventBus.ts'
  - Q Implementa la Interface    `EventBus`    q ya definimos previamente
    - Esto de q la Interface NO tenga q cambiar al modificar la Infrastructure, es 1 indicador de Interface Bien Disenada
      - Pese a q    RabbitMQ    tiene sus necesidades NO nos va a obligar a modificar la Interface, NI a meter ningun method publico q otros tengan q conocer y demas. No nos contamina el dominio ni nos obliga a acoplarnos a el


  - -- Meter     RabbitMQ    tiene mucha complejidad en la Organizacion del Codigo
    - X eso comenzaremos x usar el     "exchange"     x defecto de RabbitMQ, luego meteremos colas y demas
    - En este punto el     `addSubscribers()`    NO esta implementado xq en este video nos vamos a Centrar en la  PUBLICACION de Eventos

    -- `publish()`: Por c/evento q nos va a llegar vamos a establecer:
      - routingKeu: q va a ser el  Nombre del Evento
      - Serializamos el evento y establecemos las opciones
        - `serialize()`: Establecemos los mismos formatos q vimos en los Tests de Aceptacion
          - id: id del evento
          - type: nombre del evento
          - occurred_on: la fecha
          - attributes:  method     ".toPrimitives()"     q todos los Eventos lo Tienen
          - X ultimo lo pasamos a 1    Buffer    y eso es lo q se va a enviar
      - Las    options     son metadatos q consisten en:
        - messageId:    id del evento
        - contentType:  JSON
        - encoding:     utf-8

    -- Aqui estamos delegando bastante responsabilidad a este   `this.connection.publish`
      - Aqui estamos encapsulando la complejidad de: Serializo y se lo paso al siguiente


  - -- Creamos el     `RabbitMqConnection`     en  'src/Contexts/Shared/infrastructure/EventBus/RabbitMQ/RabbitMqConnection.ts'
    - Este es 1 Wrapper q hicieron los de   audiense   q x debajo usa la liberia   `amqplib`
      - `amqplib`  es la libreria de bajo nivel mas utilizada para conectarse a RabbitMQ

    - `connect()`: creara la conexion
      - `amqpConnect()`: envia las configuraciones de coneccion al   'connect()'
        - Aqui los Errores salen x Eventos de Node.js
          - Si hay error, haga el   reject   de la promise
      - `amqpChannel()`: abre el canal
        - Abre 1 canal con CONFIRMACION
          - `createConfirmChannel()`: Este Method va a esperar a q el     Broker     le confirme q tiene el mensaje, ANTES de resolver la promesa
            - Al final RabbitMQ funciona con ACKs, x lo q entendemos q x c/mensaje q le envias RabbitMQ te devuelve otro
              - Esto va a hacer q el intercambio de mensjes entre el Cliente y el Broker va a ser el 2ble
              - Y eso lo q va a hacer es aumentar la carga de mensajes q tiene q gestioner RabbitMQ
                - En 1 escenario de Alto Trafico, esto puede ser Muy Relevante
                  - Analogia: Protocolos de comunicacion en Red, TCP vs UDP: TCP garantiza q ha llegado todo 1 en orden mientras q en UDM se pueden perder mensajes y llegar en desorden, pero tiene > rendimiento
              - Este confirm NO te asegura q el mensaje se haya Consumido, sino q el mensaje esta ahi bien puesto
                - Es como, mi siguiente actor en el flujo de mensajes es el     broker    , una vez q el    broker    tiene el mensaje, yo ya puedo desentenderme de la responsabilidad de mantenerlo
              - X lo videos q vamos a ver luego sobre q pasa si falla la publicacion, como lo podemos gestionar, no tendria sentido utilizar 1 chanel q no nos asegura la publicacion
                - X lo general, a la hora de utilizar Eventos de Dominio, x lo general NO quieres perder ninguno
                - X lo tanto vamos a usar el   Channel   confirmando, pero teniendo en cuenta q con grandes cantidades de Trafico, eso puede suponer 1 problema importante
        - Con esto ya tenemos 1 canal de comunicacion permanente con RabbitMQ

    - Ya con esta conexion establecida a RabbitMQ podemos usar el method   `publish()`
      - Va a llamar al method  .publis()   del canar q tenemos abierto
        - Como esto no esta preparado para promises, cogemos los parametros q nos han pasado y pasarselos a la libreia de bajo nivel   "amqplib"
          - En este punto se Publica al     "exchange"    q trae x defecto RabbitMQ


    - -- Creamos el    `ConnectionSettings`    y el    `ExchangeSetting`    q son Types para tener tipado



- --- Ahora q ya lo tenemos montado, a nivel de Testing en este punto como Solo estamos PUBLICANDO, la comunicacion es unidireccional
  - X ende, el test se limita a: Mientras no pete, todo va bien

  - -- Creamos el     `RabbitMQEventBus`      en  'tests/Contexts/Shared/infrastructure/EventBus/RabbitMQEventBus.test.ts'
    - De momento Solo va a Publicar, y si publico q NO fallo, eso ya es nuestro feetback
    - En el   beforeAll()   establecemos la conexion con RabbitMQ
    - En el test, vamos a Instanciar el EvenBus y publicar 1 evento

    -- Modificamos el     `CoursesCounterIncrementedDomainEventMother`     para q tenga el Method    `create()`    q retorna 1 "DomainEvent"


    -- Agregamos   RabbitMQ   en nuestro docker-compose
      - Accedmos a:     http://localhost:15672/#/exchanges/%2F/amq.topic
        - username:   guest
        - passwords:   guest









### 🐰 Gestion de errores al publicar eventos de dominios
- --- Cuando vamos a hacer Sistemas Distribuidos se nos abre 1 mundo de posibilidades a nivel de Escalabilidad y Mantenibilidad, pero tb se nos abren ciertos retos como problemas a la hora de publicar esos eventos contra esos sistemas
  - Pueden ocurrir ciertos errores a la hora de publicar eventos en sistemas distribuidos
    - Y en este video nos enfocaremos en como reaccionar ante ello para evitar la Incongruencia de datos

  -- Algunos de los problemas podrian ser:
    - Q se Guarda en DB con exito, pero falla la publicacion del evento, con lo cual quienes escuchan no podran reaccionar a ello
      - Ej. guardas en db, y el counter NO se actualiza.
    - Q NO guarde en DB pero q SI se publique el Evento, asi los Subscribers Reaccionaran ante ello, cuando NO debieron hacerlo

  -- Para actuar antes esto tenemos:
    - Aplicar el el      Patron FailOver     con lo q tendriamos Inyectado en el Publicador de Mensajes 1 componente q Gestione los mensajes q se han intentado publicar pero q No se ha podido
    - Tb podriamos aplicar el OutBox Pattern




  - -- Modificamos el     `RabbitMQEventBus`    , solo aplicamos 1 try/cath en donde si Falla la Publicacion, llamamos al    FailoverPublisher  <--   `publih()`
    - Si falla, el   cath   lo atrapa y llama al    `.failoverPublisher()`    pasandole el Evento q se intento publicar
      - En donde este Evento es una   Entidad   de 1 Evento de Dominio


    -- Creamos el      `DomainEventFailoverPublisher`    en 'src/Contexts/Shared/infrastructure/EventBus/DomainEventFailoverPublisher/DomainEventFailoverPublisher.ts'
      - No es ninguna   Interface   xq como    RabbitMQEventBus     es 1 elemento de   Infrastructure   sus colaboradores NO necesitamos q sean Interfaces, entonces nos quitamos esa complejidad
        - De momento como Solo tenemos 1 Implementacion, no necesitamso interface
        - Pero si en el futuro Nececitas    N    implementaciones, refactorizas a Interface

      -- Este     "DomainEventFailoverPublisher"    q guarda los mensajes q se van a publicar en   MongoDB   ya q es la DB con la q estamso W
        - Estableceremos 1 conexion con Mongo, y crearemos la      Colletion     q llamaremos  'DomainEvents'
          - `publish()`: Serializaremos el Evento y guardarlo en DB
            - Aqui hacemos 1      .updateOne()      xq usamos el     {upsert: true}
              - Esto xq tenemos el     "consume()"    q podria volver a fallar y NO queremos duplicar info
                - Si vuelve a fallar  RabbitMQ  , con el      .updateOne()    actualizamos esos registros para NO duplicarlos, pero si NO existen, gracias al    upsert    los persistimos
            - Con este method guardamos en DB los mensajes q fallaron a la hora de ser Publicados con el Objetivo de q despues los recuperemos e intentemos volver a Publicarlos
          - `consule()`: Hace el   get   d esos mensajes fallidos q estan en DB en Baches de  200  para volverlos a publicar
            - Los Intentas Publicar de Nuvo a traves del       RabbitMQEventBus      , x lo tanto, q pasaria si este proceso vuelve a fallar publicando el mensaje, pues se duplicaria la data/mensajes xq volveria a entrar en el     FailOver
              - Pero gracias al      .updateOne()     cubrimos este posible error


    -- Creamos el      `DomainEventJsonSerializer`       en  'src/Contexts/Shared/infrastructure/EventBus/DomainEventJsonSerializer.ts'
      - Nos ayuda a     Serializar el Evento     para poderlo guardar en DB
      - Este es usado x el    `publish()`    del    `DomainEventFailoverPublisher`



  - -- Entonces, hasta aqui tenemos el      FailoverPattern     , pero tenemos Otro Pattern q es bastante similar y es el       OutBox Pattern      q cambia el momento en el q se guarda en DB
    -- El      `FailOver Patter`     es Reactivo ante el Error, mientras q el    OutBox Patter es Preventivo al error
    -- El      `OutBox Pattern`      lo q intenta es q el Guardado en DB de estos eventos Ocurran a la Vez q se guarda el Agregado
      - Esto mediante    Transacciones,   a la vez guardo el Agregado/Entity y el Evento pendiente de publicar
        - Este patron es mas    Preventivo del Error   xq 1ro guardas Antes de intentar Publicar
      - EXIGE q la DB tenga procesos de   TRANSACCIONALIDAD   q x lo general lo tiene todas las SQL


    -- En si, esto depende de cual confiable son los proveedores de DB y Mensajeria
      - FailOver Pattern:  Tengo mas confienza en mi DB q en el Proveedor de Mensajes RabbitMQ
      - OutBox Pattern:
        - Si usas una Solucion de CLoud q te provee el      Broker de Mensajeria     como la DB
          - Q probabilidad hay de q se caiga la 1 y el otro NO
            - Quiza la DB tenga > probabilidad de caerse q el    Broker de Mensajeria

      -- Se debe tener en cuenta esto xq meter este tipo de       `FailOver Pattern`       esta introdiciendo Mucha Complejidad tanto a la Hora de Publicacion como a la hora del   Despliegue   xq el proceso q intenta Republicar va Aparte
        - Ademas tienes q gestionar q esos mensajes NO se queden demasiado antiguos
          - X lo tanto es mucha complejidad la q estas asumiendo, y tienes q Tener Justificacion para ello
        - COmo en todo, siempre podemos tirar por soluciones mas de Infrastructure o mas de COde



  - -- Ej:
    -- Cuando tenemos 1     Monolito    y todo va contra la Misma DB, lo q puede pasar es q se Caiga la DB y Todo el Sistema para todos los CLients falle
      - En estos casos se suele Tener REPLICAS, en las q si se nos cae 1, levantamos otra
        - Aqui tenemos 1 punto central de fallo, pero almenos tenemos Replicas

    -- Con RabbitMQ podemos tener sistemas distribuidos, muchos servicios y demas, pero Todos vas a Estar Conectados contra ese RabbitMQ
      - El tema con   RabbitMQ   o cualquier sistema de mensajeria, es q SON 1 Punto Central de Fallo
        - X eso es importante tener mecanismos como este    FailOver     o replicas con  RabbitMQ y demas
        - Siempre pensar lo q se va a hacer si esto falla
          - Y siempre pensar q va a fallar en algun punto, y estar preparado para ello
      - Entonces, asi debemos preveer estas cosas e implementar soluciones, q hay varios tipos
        - Infrastructure: Tener Replicas
        - App/Code:       Este tipo de Patterns
          - Pero es mas complejo y engorroso
          - X eso, en general se aboga por Soluciones de Replicas (Infrastructure)


  -- Creamos los Tests















## ⚡ Consume Eventos desde RabbitMQ
### 🎱 Generar la configuración de RabbitMQ
- --- Ya tenemos nuestro Primer Evento en  RabbitMQ, en esa Cola x Default, pero todavia nos falta cerrar el ciclo
  - -- Vamos a hablar sobre 1 monton de cosas como:
    - La nomenclatura de los Eventos de Dominio, Como Crear Colas/Queues para c/Subscriber, etc.


  - -- Cada Subscriber va a tener su Propia Queue para que no exista cruces de competicion entre consumidores
    - Asi si se tiene q Reencolar alguna Mensaje, el Otro Consumidor/Subscriber NI se entera de todos los intentos q el Otro Consumidor haga con respecto al mismo   `RuotingKey`
    - Tambien hay que mejorar la nomenclatura del    `RuotingKey`    para q sea mas descriptivo:   Ver Drive
            `codely.mooc.1.event.course.created`


  - --



































### PRO TIP
  -- Cambiar de Repositorio de GitHub manteniendo los commits
    - Cambiar el Remoto de GitHub:

```bash
git remote set-url origin <URL del nuevo repositorio>

git remote -v

git push -u origin main
```













### Github del Example con mas cosas
  https://github.com/CodelyTV/typescript-ddd-example/tree/master/src/Contexts/Backoffice/Courses/domain







