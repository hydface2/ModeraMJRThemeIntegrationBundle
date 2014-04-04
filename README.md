# ModeraServerCrudBundle [![Build Status](https://travis-ci.org/modera/ModeraServerCrudBundle.svg?branch=master)](https://travis-ci.org/modera/ModeraServerCrudBundle)

The bundle provides a set of tools that simplifies building applications which need to operate with data coming
from client-side. These operations are supported:

 * Creating new records
 * Validating data ( both Symfony validation and domain validation )
 * Querying data - single record, batch
 * Removing record(s)
 * Getting default values that can be used on client-side as a template for a new record

What this bundle does:

 * Provides a super-type controller that you can inherit from to harness power of all aforementioned operations
 * Integrates a powerful querying language where you define queries using JSON - now you can safely build queries
   on client-side
 * Hydration package - this component provides a nice way of converting your entities to data-structure that can
   be understood by client-side logic
 * Provides a simple yet powerful client-server communication protocol
 * Simplifies functional testing of your controller

## Super-type controller essentials

As we already mentioned the bundle ships a super-type controller that you can use to spare yourself from writing your
own controllers which will be responsible for querying and persisting your data. Super-type controller
is represented by \Modera\ServerCrudBundle\Controller\AbstractCrudController class. Class defines a bunch of methods:

 * `createAction($params)` -- Method is responsible by analyzing received from client-side parameters $params create and
   persist to persistent storage a new piece of data.
 * `updateAction($params)` -- Method can be used to update existing records in persistent storage
 * `getAction($params)` -- By analyzing given $params method must return a single hydrated record
 * `listAction($params)` -- Depending on a query specified in $params may return more than one hydrated record
 * `removeAction($params)` -- Method can be used to delete record(s) from persistent storage
 * `getNewRecordValuesAction($params)` -- Method can be used to return a template data-structure that eventually
    can be used on client-side when creating a new record

For more details on these methods please see \Modera\ServerCrudBundle\Controller\CrudControllerInterface.

In order for super-type controller to work it relies on a high-level server-client communication protocol. It means
that all requests you send to server side CRUD controller must comply with the protocol. Below we are going to take
a look at each of the controller's methods and explain how you can invoke them and what structure returned data will
have.

To start using AbstractCrudController you need to create its subclass and implement `getConfig` method. For example:

    class UserCrudController extend \Modera\ServerCrudBundle\Controller\AbstractCrudController
    {
        // override
        public function getConfig()
        {
            return array(
                'entity' => 'User',
                'hydration' => array()
            );
        }
    }

    $c = new UserCrudController();
    $c->setContainer($container);

Method `getConfig` supports another dozen of configuration properties which will discuss later but `entity` and
`hydration` ones must always be provided. `entity` property must contain a fully-qualified class name of an entity
that this controller will be dealing with and `hydration` configuration property will instruct
\Modera\ServerCrudBundle\Hydration\HydrationService which is used internally by the controller how to format your data
when it is sent to client-side. Hydration process will be discussed later in great details.

Throughout this tutorial we are going to use a several dummy entities which can be represented by these classes:

    class User
    {
        public $id;

        public $username;

        public $company;

        public $creditCards = array();
    }

    class Company
    {
        public $id;

        public $name;
    }

    class CreditCard
    {
        public $id;

        public $number;
    }

In scope of this section we are not going to concern ourselves with ORM mapping.

### Error handling

The main thing you need to know that every response that comes from server side will always contain at least one boolean
property - `success`. This property can be used on client side to understand if a request sent to server side controller
was processed properly or something went wrong. When exception is thrown during processing your request an implementation
of \Modera\ServerCrudBundle\ExceptionHandling\ExceptionHandlerInterface will be used to convert the thrown exception to
a data structure that will be sent to client-side. By default \Modera\ServerCrudBundle\ExceptionHandling\EnvAwareExceptionHandler
is going to be used and depending on an environment you are working with ( dev, prod ) the returned data will look either
like this:

    // for exception like this:
    throw new \RuntimeException('oops');

    // dev
    array(
        'success' => false,
        'exception' => true,
        'exception_class' => 'RuntimeException',
        'stack_trace' => '...', // stack trace
        'file' => 'foofile.php', // full path a file where exception has occurred
        'line' => 15,
        'message' => 'oops'
    )

For production environment returned information will be more scarce:

    array(
        'success' => false,
        'exception' => true
    )

### Hydrating data

Before we go into discussions on what tools the bundle provides that allow you to hydrate you data it is worth taking
a moment and explaining what hydration is. Simply speaking, hydration it is a process of taking your in-memory
object and converting to something that can be easily send through network - yes, you can think of it as serialization,
but there's one gotcha though. Usually, when you serialize an object and then de-serialize in on the other end of the
wire then you expect to get exact or similar state the object had when it was serialized but when it comes to hydration
we naturally acknowledge that some data may be intentionally skipped, this means that a sentence "data templating" would
more accurately reflect the essence of what hydration really is - it is a process of templating data structures. For
example, a User object configured this way:

    $company = new Company();
    $company->id = 177;
    $company->name = 'Example company';

    $cc1 = new CreditCard();
    $cc1->id = 3;
    $cc1->number = 123412341324;

    $cc2 = new CreditCard();
    $cc2->id = 5;
    $cc2->number = 432143214312;

    $u = new User();
    $u->id = 5;
    $u->username = 'jane.doe';
    $u->company = $company;
    $u->creditCards = array($cc1, $cc2);

Can be hydrated in many different ways:

    array(
        'id' => 5,
        'username' => 'jane.doe',
        'company' => 177
    )

    array(
        'id' => 5,
        'username' => 'jane.doe',
        'company' => array(
            'id' => 177,
            'name' => 'Example company'
        )
    )

Internally CRUD controller relies on \Modera\ServerCrudBundle\Hydration\HydrationService class to have hydration taken
care of. HydrationService is a very flexible component providing many configuration options which must make it possible
to satisfy even the most demanding requirements.

When defining hydration configuration scheme you will be operating two configuration properties - "profiles" and "groups".
Profiles can be though of as template names, which will define how you your in-memory object will be serialized down
to a data-structure that can be sent back to client-side. Groups can be used inside profile to split data into semantic
chunks of data and later can be grouped to form profiles without duplicating code. Looking at the examples above, this
is how you could define hydration groups:

    // override
    function getConfig() {
        return array(
            'entity' => 'User',
            'hydration' => array(
                'groups' => array(
                    'form' => array('id', 'username', 'company'),
                    'list' => function(User $e) {
                        return array(
                            'id' => $e->id
                            'username' => $e->username,
                            'company' => array(
                                'id' => $e->company->id,
                                'name' => $e->company->name
                            )
                        );
                    }
                ),
                'profiles' => array(
                    'form' => HydrationProfile::create(false)->useGroups(['form']),
                    'list' => HydrationProfile::create(false)->useGroups(['list'])
                )
            )
        );
    }

If you want to use example above in your code you will need to import \Modera\ServerCrudBundle\Hydration\HydrationProfile
before you can use short class name "HydrationProfile". As you can see when we create an instance of HydrationProfile using
static factory method `create` we pass `FALSE` value which indicates that we don't want to use grouping. In our case
we have a very simple data structure, but real applications most of the time will impose more requirements - for example,
say that we have a a user profile editing window and when the window is shown we also want to show user's credit cards.
Let's update our `hydration` configuration property so it would look like this:

    // override
    function getConfig() {
        return array(
            'entity' => 'User',
            'hydration' => array(
                'groups' => array(
                    'form' => array('id', 'username', 'company'),
                    'list' => function(User $e) {
                        return array(
                            'id' => $e->id
                            'username' => $e->username,
                            'company' => array(
                                'id' => $e->company->id,
                                'name' => $e->company->name
                            )
                        );
                    },
                    'credit-cards' => function(User $e) {
                        $result = array();

                        foreach ($e->creditCards as $cc) {
                            $result[] = array(
                                'id' => $cc->id,
                                'number' => $cc->number
                            );
                        }

                        return $result;
                    }
                ),
                'profiles' => array(
                    'form' => HydrationProfile::create(true)->useGroups(['form', 'credit-cards']),
                    'list' => HydrationProfile::create(false)->useGroups(['list'])
                )
            )
        );
    }

As you can see we have updated our `form` hydration profile so it would use grouping - we passed `TRUE` to
HydrationProfile::create method and also we added `credit-cards` to `useGroups` method. Now if you query server-side
and specify hydration profile `form`:

    $result = $c->getAction(array(
        'filter' => array(
            array('property' => 5, 'value' => 'eq:5')
        ),
        'hydration' => array(
            'profile' => 'form'
        )
    ));

Then response will look like this:

    array(
        'success' => true,
        'result' => array(
            'form' => array(
                  'id' => 5,
                  'username' => 'jane.doe',
                  'company' => array(
                      'id' => 177,
                      'name' => 'Example company'
                  )
              ),
            'credit-cards' => array(
                array('id' => 3, 'number' => '123412341324' )
                array('id' => 5, 'number' => '432143214312' )
            )
        )
    )

In our client-side code we can use these groups and load data to different components - for example, `form` data will
will load a component which displays a form and data from `credit-cards` can be loaded to a grid component.

There's one another useful feature that hydration package supports out of the box. Say, that in our user profile editing
window we also want to allow to edit or add new credit cards for the user being edited. It is apparent that when we change
or add new credit-cards and changes were propagated to server-side database we want to update our UI. For this to happen
you can issue another request to server side and specify what exactly profiles you want to fetch:

    $c->getAction(array(
        'filter' => array(
            array('property' => 5, 'value' => 'eq:5')
        ),
        'hydration' => array(
            'profile' => 'form',
            'groups' => array('credit-cards')
        )
    ));

When you use 'groups' configuration property in request then hydration package will detect it and will send only those
groups from hydration profile that you have specified, this allows to make client-server communication process more
effective.

So far we have seen how to define your hydration groups using two approaches:

* Using array with expressions
* Anonymous function

Most of the time these two approaches will suffice but sometime you mays want to create a reusable hydration group
that you probably will want to re-use in several controllers. Luckily, hydration package can be easily extended.
Essentially when you define your hydration group you can pass anything that PHP can treat as callable - anonymous function,
array, a class which implements special \__invoke method. This is how reusable credit card hydrator could look like:

    class CreditCardsHydrator
    {
        public function \__invoke(User $u, ContainerInterface $container)
        {
            $result = array();

            foreach ($e->creditCards as $cc) {
                $result[] = array(
                    'id' => $cc->id,
                    'number' => $cc->number
                );
            }

            return $result;
        }
    }

Whenever HydrationService handles a hydration group it will pass instance of Symfony ContainerInterface as a last
parameter, this will prove useful when you have complex hydration logic.

In simple cases it might happen that your hydration group will have one-to-one relation with hydration profile, in this
case when defining profiles you may not use HydrationProfile and instead just specify a group name, for example:

    // override
    function getConfig() {
        return array(
            'entity' => 'User',
            'hydration' => array(
                'groups' => array(
                    'list' => function(User $e) {
                        return array(
                            'id' => $e->id
                            'username' => $e->username,
                            'company' => array(
                                'id' => $e->company->id,
                                'name' => $e->company->name
                            )
                        );
                    },
                ),
                'profiles' => array(
                    'list'
                )
            )
        );
    }

When hydration profiles are defined using this syntax then no result grouping will be used.

Also, you when defining composite hydration profiles you may using a short-hand syntax without using HydrationProfile:

    // override
    function getConfig() {
        return array(
            'entity' => 'User',
            'hydration' => array(
                'groups' => array(
                    'form' => array('id', 'username', 'company'),
                    'credit-cards' => function(User $e) {
                        $result = array();

                        foreach ($e->creditCards as $cc) {
                            $result[] = array(
                                'id' => $cc->id,
                                'number' => $cc->number
                            );
                        }

                        return $result;
                    }
                ),
                'profiles' => array(
                    'form' => ['form', 'credit-cards']
                )
            )
        );
    }

In this case grouping will be used and invocation results of groups `form` and `credit-cards` will grouped under
under group names.

### Querying data

As we already mentioned the super-type controller provides two actions you can use to fetch data from server:
`getAction($params)` and `listAction($params)`. The main difference is that if a query provided in $params when `getAction`
is invoked returns more than one record then exception will be thrown. When either of methods is invoked you must
use `filter` request property which will contain a query that server can use to find records that it
must return and also a `hydration` property.

To fetch a single record from server-side you need to use `getAction($params)` method, provided query must have conditions
which will make it return exactly one record, usually you will use a primary key. Example:

    $result = $c->getAction(array(
        'filter' => array(
            array('property' => 'id', 'value' => 'eq:5')
        ),
        'hydration' => array(
            'profile' => 'form'
        )
    ));

And this is how we can use `listAction($params)`:

    $result = $c->getAction(array(
        'filter' => array(
            array('property' => 'username', 'value' => 'like:%doe%')
        ),
        'hydration' => array(
            'profile' => 'form'
        )
    ));

Internally to analyze `filter` property ModeraServerCrudBundle and build DQL the bundle relies on
[SliExtJsIntegrationBundle](https://github.com/sergeil/SliExtJsIntegrationBundle) bundle. Please consult to its
documentation to learn how to to structure your queries. It it is worth saying that without exaggerating that we
SliExtJsIntegrationBundle is used be able to build up to 90% of your queries right on client-side without writing DQL
manually on server-side. The bundle has support for working and filtering by associations, applying database functions
and using grouping, pagination amongst others.

### Creating new records and updating records

When you send a request to update a record or create a new one all your record data must be consolidated under associative
property `record`. For example, to create a new record you can invoke `createAction($params)` method by passing parameters
similar to these ones:

    $result = $c->createAction(array(
        'record' => array(
            'username' => 'john.doe',
            'company' => 5
        ),
        'hydration' => array(
            'profile' => 'form'
        )
    ));

In order to update already existing method you need to use `updateAction($params)` method and you must specify primary
keys' values under `record` property so the server side can identity what exactly record it must update:

    $result = $c->updateAction(array(
        'record' => array(
            'id' => 5,
            'company' => 71
        ),
        'hydration' => array(
            'profile' => 'form'
        )
    ));

Aside from conventional `success` property responses may also optionally contain associative `result` property
( this depends on either `hydration` request parameter was provided or not, this is going to be discussed later
Hydrating data section ). Also, whenever a new record is created or existing one got updated, server-side may add
any of these properties to response:

* created_models:
* updated_models
* removed_models

These properties will contain model names and IDs of records that were affected while processing your request. Later you
can use these properties on client-side to update your UIs. For example, when we issued user update request above then
response from server-side could have looked akin to the following:

    array(
        'success' => true,
        'updated_models' => array(
            'sample_user' => [5]
        )
    )

The way how server-side fully qualified entity class name gets converted to something probably short which we use
one client-side is handled by implementations of \Modera\ServerCrudBundle\Persistence\ModelManagerInterface, this topic
will be discussed later.

Quite often when a new record is being created or existing one is getting updated some validation errors may occur.
Protocol distinguishes two types of validation errors: general errors and fields associated errors. To
categorize validation errors we can use an example. For example, if user doesn't have required permissions to create a new
record, then this validation error will rather fall to "general errors' category but if for a field email user entered
a non-valid email address then this validation error will rather belong to "fields associated errors" group. Server
responses for these types of errors will look similar to examples shown below.

General validation error:

    array(
        'success' => false,
        'general_errors' => array(
            'You do not have require privileges to create a new user records!'
        )
    )

Field validation error:

    array(
        'success' => false,
        'field_errors' => array(
            'email' => array(
                'Please provide a valid email address'
            )
        )
    )

Each field validation group may contain several errors.

### Removing records

To remove one or more records you will need to use `removeAction($params)` method, request must contain at least one
property - `filter`, this property must define a query that will define what records must be deleted from persistent
storage, for example:

    $c->removeAction(array(
        'filter' => array(
            array('property' => 'username', 'like' => '%doe%')
        )
    ));

If you want to delete all records then you need to provide an empty `filter` request parameter.

### Getting new record data structure template

Sometime when you create a new record, on client-side you may want to use some meaningful default values. To address
this requirement AbstractCrudController has a special method `getNewRecordValuesAction`. Internally method will
use \Modera\ServerCrudBundle\NewValuesFactory\DefaultNewValuesFactory. This class will try to find method `formatNewValues`
on an entity that given CRUD controller is configured to work with and if the method is found then it will be invoked and
values returned by this method will be sent back to client side. The method will receive 3 parameters:

* array $params -- Parameters that were passed to controller's `getNewRecordValuesAction($params)` method
* array $config -- Configuration parameters of the crud ( value returned by AbstractCrudController::getPreparedConfig() )
* ContainerInterface $container -- Instance of Symfony dependency injection container

If we would want that whenever a new user is created from client-side to select some company for it, then we could
have updated our User entity to look like this:

    class User
    {
        public $id;

        public $username;

        public $company;

        public $creditCards = array();

        static public function formatNewValues()
        {
            return array(
                'company' => 123
            );
        }
    }

## Advanced usage

AbstractCrudController class uses many different services to process your requests and if it is needed you can easily
write your own implementations to change its logic - for example, you may want to switch from Doctrine based persistence
layer and instead use a remote data source like web-services - from a day one the architecture has been designed to be
extensible and susceptible for modifications. In this section we are going to shed some light on base interfaces that
AbstractCrudController relies upon.

### Mapping data

If you have very specific requirements when it comes to binding data from client-side onto your entities then you can
extend existing or create a new implementation of \Modera\ServerCrudBundle\DataMapping\DataMapperInterface interface.
Once you have created it you can use bundle semantic configuration to register it:

    modera_server_crud:
        data_mapper:  your_mapper_di_service_id

When using bundle semantic configuration then all subclasses of \Modera\ServerCrudBundle\Controller\AbstractCrudController
will use a new data-mapping logic but sometimes all you need to is to have non-standard mapping logic just for one
specific controller, in this case you can use `map_data_on_create` and `map_data_on_update` configuration properties
when implementing `getConfig` method:

    // override
    public function getConfig() {
        $fn = function(array $params, $entity, DataMapperInterface $defaultMapper, ContainerInterface $container) {
            $defaultMapper->mapData($params, $entity);

            $sc = $container->get('security.context');

            $user = $sc->getToken()->getUser();
            if (is_object($user)) {
                $entity->setManager();
            }
        }

        return array(
            'entity' => '...',
            'hydration' => '...',
            'map_data_on_create' => $fn,
            'map_data_on_update' => $fn
        );
    }

Both `map_data_on_create` and `map_data_on_update` share identical method signature ( set of accepted parameters ).

### Creating new instances of entities

Before something can be persisted to database you need to convert a data-structure received from client-side to something
that you can pass to persistence layer. When you are working with ORM ( because you are using Symfony we believe you are
using ORM ) before you can map data onto entity you need to create its instance of configured with `getConfig` method
configuration property `entity`. AbstractCrudController relies on implementation of
\Modera\ServerCrudBundle\EntityFactory\EntityFactoryInterface to have new entities created. By default a very simple
implementation is used which will create instances using class constructors -
\Modera\ServerCrudBundle\EntityFactory\DefaultEntityFactory. Sometimes you may want to add support for class factory methods,
for this to happen you can implement EntityFactoryInterface and register your implementation using `entity_factory`
bundle configuration parameter:

    modera_server_crud:
            entity_factory:  your_entity_factory_service_id

If you don't want to make all subclasses of AbstractCrudController use your service but instead just one specific controller
then you can use `create_entity` configuration property when implementing `getConfig` method:

    // override
    public function getConfig() {
        return array(
            'entity' => '...',
            'hydration' => '...',
            'create_entity' => function(array $params, array $config, EntityFactoryInterface $defaultFactory, ContainerInterface $container) {
                return EntityClass::create();
            }
        );
    }

### Handling exceptions

Sometimes your application logic may throw domain exception that you want to handle in a very specific way or a response
that you want to send to server-side should convey some additional data. If you have this requirement then you need
to use `exception_handler` bundle configuration property:

    modera_server_crud:
        exception_handler: your_handler_service_id

`your_handler_service_id` must point a DI service which implements \Modera\ServerCrudBundle\ExceptionHandling\ExceptionHandlerInterface
interface.

Approach described above will change logic for all controllers, if you need to change logic for just one controller
then you can use `exception_handler` configuration property when implementing `getConfig` method:

    // override
    public function getConfig() {
        return array(
            'entity' => '...',
            'hydration' => '...',
            'exception_handler' => function(\Exception $e, $operation, ExceptionHandlerInterface $defaultHandler, ContainerInterface $container) {
                $response = $defaultHandler->createResponse($e, $operation);
                if ($e instanceof TranslataleException) {
                    $response['message'] = $container->get('translator')->translate($e->getToken());
                }

                return $response;
            },
        );
    }

`$operation` parameter of 'exception_handler' callback will contain values of ExceptionHandlerInterface::OPERATION_*
constants.

### Create data templates for new records

As we already explained in "Getting new record data structure template" section sometimes you may need some default
values for fields when create a new record on client-side side. A component which is responsible for this logic is configured
using `new_values_factory` bundle configuration property:

    modera_server_crud:
        new_values_factory: service_id

`service_id` must point to a service which implements \Modera\ServerCrudBundle\NewValuesFactory\NewValuesFactoryInterface
interface.

If don't want to create a universal handler for all controllers and want to change default logic for just one controller
then you need to use `format_new_entity_values` configuration property when implementing `getConfig` method:

    // override
    public function getConfig() {
        return array(
            'entity' => '...',
            'hydration' => '...',
            'format_new_entity_values' => function(array $params, array $config, NewValuesFactoryInterface $defaultImpl, ContainerInterface $container) {
                $now = new \DateTime('now');
                return array(
                    'billing_date' => $now->format('d.m.Y');
                );
             },
        );
    }

### Persistence and querying

Persistence is a term which is used to generally refer to a layer which is responsible for persisting your data to
some non-ephemeral storage - once you pushed your data to persistence storage later you should be able to fetch from there.
Most of the time when you think of persistence you will think of relational databases like MySQL, or NoSQL databases
like MongoDB but essentially you can store your data in a plain file or even use some remote web-service endpoint,
AbstractCrudController doesn't need to know details because it relies on
\Modera\ServerCrudBundle\Persistence\PersistenceHandlerInterface interface not on any specific implementation. Out of the
box the bundle provides implementation which is capable of persisting data to all databases which Doctrine ORM supports,
but if you need to create your own implementation then you will need to implement PersistenceHandlerInterface, register
it in service container and use bundle's configuration property `persistence_handler`:

    modera_server_crud:
        persistence_handler:  my_persistence_handler_service

If all you need is to apply some additional conditions on how to data gets saved to persistent storage or updated, then
you can use `save_entity_handler` or `update_entity_handler` when implementing `getConfig()` method:

    // override
    public function getConfig() {
        $fn = function($entity, array $params, PersistenceHandlerInterface $defaultHandler, ContainerInterface $container) {
            $container->get('logger')->info(sprintf('Persisting %s to database', get_class($entity));

            $defaultHandler->
        }

        return array(
            'entity' => '...',
            'hydration' => '...',
            'save_entity_handler' => function($entity, array $params, PersistenceHandlerInterface $defaultHandler, ContainerInterface $container) {
                 $container->get('logger')->info(sprintf('Persisting %s to database', get_class($entity));

                 $defaultHandler->save($entity);
             },
            'update_entity_handler => function($entity, array $params, PersistenceHandlerInterface $defaultHandler, ContainerInterface $container) {
               $container->get('logger')->info(sprintf('Updating %s in database', get_class($entity));

               $defaultHandler->update($entity);
            },
        );
    }

As you can see `save_entity_handler` and  `update_entity_handler` share same arguments set.

### Validating

Having your data validated before it gets persisted is inevitable step that you need to consider implementing to guarantee
data consistency. When it comes to validation AbstractCrudController relies on implementations of
\Modera\ServerCrudBundle\Validation\EntityValidatorInterface interface to process validation. Default implementation which
is represented by \Modera\ServerCrudBundle\Validation\DefaultEntityValidator class allows you to leverage all power
of built-in symfony validation package and adds one extra thing on top - domain validation. When implementing
`getConfig()` method you can use `ignore_standard_validator` and `entity_validation_method` configuration properties.
By default, whenever an entity gets persisted DefaultEntityValidator will check if `ignore_standard_validator` is
still set to `FALSE` ( it's default value is FALSE ) and everything's okay it will try to locate method name which
is configured with `entity_validation_method` ( default method name is `validate` ) and invoke this to let entity do some
additional domain validation. For example, if we have a User entity which has $address field and before entity is
persisted we want to make sure that provided address really exists, then we could come up with something like this:

    class User
    {
        public $address;

        public function validate(ValidationResult $result, ContainerInterface $c)
        {
            if (!$c->get('get_service')->addressExists($this->address)) {
                $result->addFieldError('address', "Given address doesn't seem to exist");
            }
        }
    }

First argument passed to `validate` method is instance of \Modera\ServerCrudBundle\Validation\ValidationResult and
must be used to report validation errors - you can report both field related errors as well as general ones.

### Actions intercepting

Sometimes you may want to add apply some additional logic before some controller actions get executed. There could many
use cases - you may need to add security enforcement or some logging logic, to name a few. In order to apply some additional
code before any of web-exposed AbstractCrud controller actions gets executed you need to do to these two simply steps:

* Create an implementation of \Modera\ServerCrudBundle\Intercepting\ControllerActionsInterceptorInterface interface
or subclass \Modera\ServerCrudBundle\Intercepting\ControllerActionsInterceptor if you don't want to write boilerplate
code.
* Register an instance of \Sli\ExpanderBundle\Ext\ContributorInterface which would return an instance of your interceptor
in a dependency injection container and tag it with "modera_server_crud.intercepting.cai_providers".

### Security

Quite often you will need to secure your controllers. All your subclasses can be secured by using "security"
configuration key when overriding `AbstractCrudController::getConfig` method:

    // override
    public function getConfig()
    {
        return array(
            // ...
            'security' => array(
                'role' => 'ROLE_ACCESS_USERS',
                'actions' => array(
                    'create' => 'ROLE_CREATE_USER'
                )
            )
        );
    }

When `security/role` configuration property is provided then user must have this role in order to access all controller
actions. If you need to add more fine-grained security role requirements then you need to use `security/actions` property,
each key of this array corresponds to a controller actions name without "Action" suffix and its value is a security
role name that user must have in order to invoke this action.