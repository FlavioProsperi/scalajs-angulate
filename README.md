Scala.js Bindings for AngularJS
===============================

**News**: angulate 0.2.1 has been released! Please consult the [release notes](https://github.com/jokade/scalajs-angulate/wiki/Release-Notes#version-021-130615) for a list of changes since version 0.1.
There is also a page with [currently known problems & limitations](https://github.com/jokade/scalajs-angulate/wiki/Known-Problems).

This library only provides bindings to Angular 1.x. Support for [Angular 2](https://angular.io) is provided by [angulate2](https://github.com/jokade/angulate2).

##### Contents:
* [SBT settings](#sbt-settings)
* [Modules](#defining-a-module)
* [Controllers](#controllers)
* [Dependency Injection](#dependency-injection)
* [Services](#services)
* [Directives](#directives)
* [Other enhancements](#other-enhancements)
  * [HttpService](#httpservice)
  * [Components](#components)

Introduction
------------
**scalajs-angulate** is a small library to simplify developing [AngularJS](http://angularjs.org/) applications in [Scala](http://www.scala-lang.org) (via [Scala.js](http://www.scala-js.org)). To this end it provides:

*  [façade traits](http://www.scala-js.org/doc/calling-javascript.html) for the Angular core API
*  macros for defining controllers, services and directives in a more natural Scala style.

There is a [complete example](https://github.com/jokade/scalajs-angulate-todomvc) implementing the TodoMVC with scalajs-angulate.

How to Use
----------

### SBT Settings
Add the following lines to your ```sbt``` build definition:
```scala
libraryDependencies += "biz.enef" %%% "scalajs-angulate" % "0.2.1"
```
scalajs-angulate 0.2.1 requires Scala.js `0.6.3+`.

If you want to test the latest snapshot, change the version to `0.3-SNAPSHOT` and add the Sonatype snapshots repository to your `build.sbt`:
```scala
resolvers += "Sonatype OSS Snapshots" at "https://oss.sonatype.org/content/repositories/snapshots"
```

### Defining a Module

```scala
import biz.enef.angulate._
import biz.enef.angulate.core.HttpService
import biz.enef.angulate.ext.{Route, RouteProvider}

val module = angular.createModule("app", Seq("ui.bootstrap","ngRoute"))

module.serviceOf[UserService]
// - or, setting the service name explicitly:
// module.serviceOf[UserService]("uservice")

module.controllerOf[UserCtrl]
// - or, setting the controller name explicitly:
// module.controllerOf[UserCtrl]("Users")

module.directiveOf[UserDirective]
// - or, setting the directive name explicitly:
// module.directiveOf[UserDirective]("person")

module.config( ($routeProvider:RouteProvider) => {
  $routeProvider.
    when("/user/:id", Route( templateUrl = "/tpl/userDetails.html" )).
    otherwise( Route( redirectTo = "/" ) )
})

module.run( initApp _ )

def initApp($http: HttpService) = ...
```

### Controllers
scalajs-angulate provides macros to allow using plain scala classes as Angular controllers.
Two flavors are currently supported:
*  [Body Scope controllers](#body-scope-controllers), extending `Controller`
*  [Controllers with explicit scope](#controllers-with-explicit-scope), extending `ScopeController`

Both flavors support constructor [dependency injection](#dependency-injection).

#### Body Scope Controllers
Classes extending `Controller` export all their public `var`s, `val`s and `def`s into the controller scope.
Defining a custom `Scope` type is not required. However, instantiating the controller in the template requires the new AngularJS `as` syntax:

```scala
val module = angular.createModule("counter")
module.controllerOf[CounterCtrl]
  
class CounterCtrl extends Controller {
  var count = 0
    
  def inc() = count += 1
    
  def dec() = count -= 1
    
  // private properties and functions are not exported to the controller scope
  private def foo() : Unit = ...
}
```
```html
<html ng-app="counter">
  <body>
  <div ng-controller="App.CounterCtrl as ctrl">
  Count: {{ctrl.count}} <button ng-click="ctrl.inc()">+</button> <button ng-click="ctrl.dec()">&ndash;</button>
  </div>
  
  <!-- ... -->
  
  </body>
</html>
```

If you need access to the AngularJS `$scope` object, just inject it into the constructor:
```scala
class Ctrl($scope: Scope) extends Controller {
  $scope.$watch( /* ... */ )
}
```
_Note_: the `$scope` injected into the controller in the example above, is _not_ the scope avaliable in the template and does not have the public members of the controller class. So, if you want to watch on a member defined in your controller, you'll have to do it this way:
```scala
class Ctrl($scope: Scope) extends Controller {
  var count = 0
  
  $scope.watch( () => count, (cnt: Int) => 
    /* code to be executed when count changes (don't forget to $digest...) */
  )
}
```

#### Controllers with explicit Scope
Classes extending `ScopeController` are "old-style" AngularJS controllers, where the scope available in the template must be injected explicitly:

```scala
val module = angular.createModule("counter")
module.controllerOf[CounterCtrl]
  
/* Option A: using a custom defined Scope type */
trait CounterScope extends Scope {
  var count : Int = js.native
  var inc : js.Function = js.native
  var dec : js.Function = js.native
}

class CounterCtrl($scope: CounterScope) extends ScopeController {
  $scope.count = 0
  
  $scope.inc = () => $scope.count += 1
  
  $scope.dec = () => $scope.count -= 1
}
  
/* Option B: without explicit Scope type (using js.Dynamic instead) */
class DynamicCounterCtrl($scope: js.Dynamic) extends ScopeController {
  $scope.count = 0
    
  $scope.inc = () => $scope.count += 1
    
  $scope.dec = () => $scope.count -= 1
}
```
```html
<html ng-app="counter">
  <body>
  <div ng-controller="App.CounterCtrl">
  Count: {{count}} <button ng-click="inc()">+</button> <button ng-click="dec()">&ndash;</button>
  </div>
  
  <!-- ... -->
  
  </body>
</html>
```


### Dependency Injection
scalajs-angulate supports constructor dependency injection of Angular services into controllers, services and directives:

```scala
class UserCtrl($http: HttpService) extends Controller {
  /* ... */
  
  $http.get("/rest/users").onSuccess{ res => ... }
}
```
the Angular `$http` service will be injected during Controller instantiation; no annotations or additional traits are required, as long as the parameter name in the constructor matches the name of the service to be injected (don't worry about JS minification, the macro translates the constructor into a String-based DI array).

If you cannot or don't want to use the service name as parameter name, you can define the service to be injected explicitly with the `@named` annotation:
```scala
class UserCtrl(@named("$http") httpService: HttpService) extends Controller {
  /* ... */
}
```


DI is also supported for functions passed to methods such as `Module.config()` and `Module.run()`:
```scala
module.config( ($routeProvider: RouteProvider) => {
  /* ... */
})

// -- or --
def routing($routeProvider: RouteProvider) = $routeProvider.when( /* ... */ )

module.config( routing _ )
```
However, the `@named` annotation is not supported for function DI, i.e. the _parameter names must match the services_ to be injected for this to work.

Functions are implicitely converted to AnnotatedFunction which transform the function into an array following the [Inline Array Annotation](https://docs.angularjs.org/guide/di).
You can use this to declare the function in one place and use it in another:
```scala
val configFn: AnnotatedFunction = ($logProvider: js.Dynamic) => {
  /* ... */
}

val module = angular.createModule("app", Nil, configFn)
```

### Services
Services can be implemented as plain classes extending the `Service` trait. As with controllers,
constructor based dependency injection is supported:
```scala
class UserService($http: HttpService) extends Service {
  def getUsers() : HttpPromise = $http.get("/rest/users/")
}

// registers the service with the name 'userService'
module.serviceOf[UserService]

// -- or --

// registers the service with the name 'users'
module.serviceOf[UserService]("Users")


class UserCtrl(userService: UserService) extends Controller {
  /* ... */
}
```
If no explicit service name is provided to `serviceOf[Service]`, then the class name will be used as service name,
__with the first letter in lower case__ (to support derivation of dependencies from argument names, which begin with a lower case letter by convention).


### Directives
__Note:__ This section describes the directive API provided by scalajs-angulate 0.2.

To implement an AngularJS directive, create a class extending `Directive` and override all members you want to define on the [directive definition object](https://docs.angularjs.org/api/ng/service/$compile):
```scala
class HelloUserDirective($animate: AnimationService) extends Directive {
  // the type of the scope object passed to postLink() and controller()
  override type ScopeType = js.Dynamic
  
  // the type of the controller instance passed to postLink() and controller()
  override type ControllerType = HelloUserCtrl
  
  override val restrict = "E"
  
  override val transclude = false
  
  override val template = """<div>Hello {{user}}><div>"""
  // -- or --
  // override def template(element,attrs) = ...
  // -- or --
  // override val templateUrl = "/url"
  // -- or --
  // override def templateUrl(element,attrs) = ...
  
  override val isolateScope = js.Dictionary( "user" -> "@" )
  // -- or --
  // override val scope = true
  
  override def postLink(scope: ScopeType,
                        element: JQLite,
                        attrs: Attributes,
                        controller: ControllerType) = ...
                        
  // override def compile(tElement: js.Dynamic, tAttrs: Attributes) : js.Any = ...
}

// defines the directive under the name "helloUser"
module.directiveOf[HelloUserDirective]
// -- or --
// module.directiveOf[HelloUserDirective]("sayHello")
```

If you need access to the AngularJS `$compile` function, just inject it into the directive constructor:
```scala
class MyDirective($compile: Compile) extends Directive {
  override def postLink(scope: ScopeType,
                        element: JQLite,
                        attrs: Attributes,
                        controller: ControllerType) : Unit = {
    val tpl = $compile( /* ... */ )
    /* ... */
  }
}
```

If you need to define properties or methods on the directive controller or scope, you can do so in the `controller` method:
```scala
trait UserDirectiveCtrl extends js.Object {
  var greet: js.Function = js.native
}

class UserDirective extends Directive {
  override type ScopeType = js.Dynamic
  
  override def controller(ctrl: ControllerType,
                          scope: ScopeType,
                          elem: JQLite,
                          attrs: Attributes) : Unit = {
    scope.greeting = "Hello"
  }
}
```

### Other enhancements
This section gives an overview over the enhancements to AngularJS core modules provided by angulate.

#### HttpService
The API of the AngularJS `$http` service is provided by the trait `biz.enef.angulate.core.HttpService`.
The `get`, `put`, `post` and `delete` functions on this trait are enhanced with a type parameter that allows to specific the expected return type of the response. Furthermore, these functions return an instance of `HttpPromise[T]` which is a representaion Angular's http promise object with the following additional functions:
```scala
trait HttpPromise[T] extends js.Object {
  /* ... (API provided by AngularJS) */
  
  def onComplete(f: Try[T] => Unit) : HttpPromise[T]
  
  def onSuccess(f: T => Unit) : HttpPromise[T]
  
  def onFailure(f: (HttpError) => Unit) : HttpPromise[T]
  
  def map[U](f: T => U) : HttpPromise[U]
  
  // transforms this HttpPromise into a standard Scala Future
  def future: Future[T]
}
```
Calls to `onComplete`, `onSuccess`, and `onFailure` are translated into calls to `success` and `error` of the AngularJS HttpPromise API.

###### Example:
```scala
trait User extends js.Object {
  var id: Int = js.native
  var name: String = js.native
}

class UserService($http: HttpService) extends Service {
  def getAll() : HttpPromise[js.Array[User]] = $http.get("/users")
}

class UserCtrl(userService: UserService) extends Controller {
  var users = js.Array[User]()
  
  readAll()
  
  def readAll() : Unit = userService.getAll onComplete {
    case Success(data) => users = data
    case Failure(ex) => handleError(ex)
  }
  
  def handleError(ex: Throwable) = ...
}
```

#### Components
__EXPERIMENTAL__
Starting with version `0.2`, Angulate has limited support for [Angular2-style components](http://angular2.com/get-started.html#). An angulate component is just a plain scala class annotated with `@Component()`; all public members defined in the class are then accessible from the component's template:
```scala
import biz.enef.angulate._

@Component(ComponentDef(
  selector = "counter",  // component name (i.e. the HTML tag)
  template = """{{count}} <button ng-click="inc()">+</button> <button ng-click="dec()">-</button>""",
  // - or -
  // templateUrl = "counter.html"
  bind = js.Dictionary(
   "init" -> "@"  // assign the value of the DOM attribute 'init' to the class property with the same name 
  )
))
class Counter {
  
  var count = 0
  
  def init = ???
  // called with the value of the DOM attribute 'init'
  def init_=(s: String) = count = s.toInt
  
  def inc() = count += 1
  
  def dec() = count -= 1
  
}

val module = angulate.createModule("foo")
module.componentOf[Counter]
```
The component can then be used as a custom HTML tag:
```html
<counter init="42" />
```

License
-------
This code is open source software licensed under the [MIT License](http://opensource.org/licenses/MIT)

