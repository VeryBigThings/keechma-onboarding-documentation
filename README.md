# Keechma
Keechma is a micro framework for Reagent written in ClojureScript. It implements a layered set of utilities, isolated from each other, that makes it easier to build a deterministic and predictable application. More info can be found at https://keechma.com/guides/.

#### Note 
Keechma Framework is pretty agnostic when it comes to implementation, but for our day-to-day work, we've developed [Keechma Toolbox](https://github.com/keechma/keechma-toolbox) which we use extensively. 
The most used tools are - Pipelines, Dataloader, UI (and CSS) helpers and Forms*. Since the purpose of this document is onboarding new people to existing projects we will cover them also.

\* _Keechma Forms are not coupled with the rest of the Keechma ecosystem and can be used in any Reagent based application._

## Basics
In it's basic form Keechma Application consists of 4 layers that work in a sequential order - [Router](https://keechma.com/guides/router/), [Controllers](https://keechma.com/guides/controllers/), [AppDB*](https://keechma.com/guides/entitydb/) and the [UI layer](https://keechma.com/guides/ui-system/).

1. **The router** converts the URL to the data
2. The controller manager receives the URL data and orchestrates **the controllers**
3. Controllers react to the route data change and mutate the **AppDB**
4. AppDB changes are reflected in the **UI**

This makes data flow unidirectional and predictable, which makes a complex system easier to build and manage.

\* _AppDB is an internal atom, a place where applications state is stored. EntityDB is its subset which stores and maintains loaded data (entities)_

Feel free and visit the following links to find out more about [how](https://keechma.com/guides/introduction/) and [why](https://keechma.com/news/route-driven-apps-rock/) of Keechma.


## ROUTER
Route processing is one of the key parts of Keechma and to have a better understanding of how UI layer gets rendered I strongly recommend taking a look at ["Route driven apps rock"](https://keechma.com/news/route-driven-apps-rock/) and ["Building Hello world app" ](https://keechma.com/news/hello-world-app-in-keechma-with-routing/) blog posts.

In short - _Keechma treats the URL as the single source of the truth. Application state is derived from the route params._

What that means is that every time the route changes it will trigger a chain of the events that will produce the UI you're presented within the end. That chain of the events follows the 4 aforementioned layers: 

Route changes **=>** Controller manager decides which controller to start/stop/restart **=>** those controllers in turn mutate AppDB **=>** UI displays the new data

Unlike other systems, you don't associate an action with a route. Each URL change will result in the following:

1. URL will be transformed into a params map
2. Route params will be sent to the Controller Manager
3. The Controller Manager will start or stop controllers according to the route params


Route data is stored in the application state (it can be deserialized or the route patterns can be defined) and the route can be accessed via subscriptions.

If we don't define route patterns and some data is stored in the URL (_ie_ `?name=John`), the returned route map will look like this:

```
{:data {:name "John"}}
```

We could define route patterns in `core.cljs` like this:

```
(def pretty-route-app-definition
  {:components   {:main main-component}
   :html-element (.getElementById js/document "app")
   :routes       [":name"]})
```

which would in return match first param passed after `/` to `:name` and route like this `/John` would look like the same in returned route map:

```
{:data {:name "John"}}
```

Many additional patterns and options are available, including setting default values to params:

``` 
[["" {:name "Student"}]                ;; this pattern will match an empty route (ie first app load), and will set the `:name` param to "Student"
 ["name/:name" {:name "Student"}]]     ;; this pattern will match any route that starts with `name/` even if the :name param is not set - in that case it will set the :name to "Student"
```

Custom `route-processor` function can be passed to the apps core configuration file which allows you to intercept and modify route data if needed.

## CONTROLLERS
Controllers in Keechma react to route changes and implement any code that has side effects. They are the only place in the application where you have access to the application's state atom. You can have as many controllers as you want which allows you to split applications logic in as many parts as necessary. 

A detailed explanation of how controllers work can be found [here](https://keechma.com/guides/controllers/), but most important takeaways are:
* `params` function is the main driver of change - it receives `route` params and returns a subset of the params needed for the controller to run  (or `nil`)
* based on `params` return value **Controller Manager** decides if that controller should be started, stopped, restarted or left alone
* they have a number of implemented functions
* they can react to users commands sent from UI layer

### Basics
Controllers are `:required` from `keechma.controller`
```
(ns foo.controllers.some-controller
  (:require [keechma.controller :as controller]
             ...)
  (:require-macros [cljs.core.async.macros :refer [go-loop]]))
```

We define controller as a record:

```
(defrecord SomeController [])
``` 

and we make use of its `params` function as follows:

```
(defmethod controller/params SomeController [_ route-params]
  (get-in route-params [:data :name])
```

Besides `params` function, most commonly used actions are `start` and`stop` functions (which get called when controllers are started or stopped) and the `handler` function.

`Handler` function is used to receive user commands from the UI layer. It listens for commands on the `in-chan` channel inside the `go` block and looks something like this:

```
(defmethod controller/handler SomeController [_ app-db-atom in-chan _]
    (go-loop []
      (let [[cmd _] (<! in-chan)]
        (when cmd
          (when (= :some-command cmd) (do-something))
          (recur)))))
```

Controllers can be registered in two ways (stating both of them for clarity reasons):

##### NEWER
inside `some-controller.cljs` file:
```
(defn register
  ([] (register {}))
  ([controllers] (assoc controllers :some-controller (->Controller))))
```
and declared in the `controllers.cljs` file:
```
(ns site.controllers
  (:require [site.controllers.some-controller :as some-controller]))

(def controllers
  (-> {}
      some-controller/register))
```
##### or OLDER 
(without register fn)

```
(ns site.controllers
  (:require [site.controllers.some-controller :as some-controller]))

(def controllers
  (-> {:some-controller (some-controller/->Controller)}))
```

More on why Keechma Controller can be read [here](https://keechma.com/news/why-keechma-controllers/), and while this low-level abstraction gives you the power to solve problems without having to force-fit them to our framework, they produce a lot of boilerplate code so we came up with another abstraction on top of them that makes simple (and often repeated) problems easy to solve - `Pipelines` 

### Pipelines (Keechma Toolbox)

[Pipelines](https://keechma.com/news/introducing-keechma-toolbox-part-1-pipelines/) embrace the asynchronous nature of frontend development while allowing you to keep the related code grouped together. 

Relatively common pattern of loading some data from the server, informing the user about it and handling of the potential errors can easily be modeled by pipelines with something like this:

```
(pipeline! [value app-db]
    (pp/commit! (assoc app-db :articles-status :loading))          ;; Mark status as `loading`
    (load-articles-from-server)                                    ;; Load some articles
    (pp/commit! (-> app-db
                  (assoc :articles-status :loaded)
                  (assoc :articles value)))                        ;; Once the articles are loaded, mark the status as `loaded` and store them in `app-db`
   (rescue! [error]
     (pp/commit! (assoc app-db :articles-status :error))))         ;; In case of an error, mark the status as `error`
```

To make this work, their implementation makes some ground rules to follow:

1. Pipelines are built from a list of functions
2. Each function can be either a side-effect or a processor function
3. `value` is bound to the command arguments or the return value of the previous processor function
4. `app-db` value is always bound to the current state of the app-db atom
5. Side-effects can't affect the `value` - their return value is ignored
6. If a processing function returns a promise, pipeline will wait until that promise is resolved or rejected before proceeding to the next function
7. If a processing function returns `nil`, the `value` argument will be bound to the previously returned value
8. Any exception or promise rejection will cause the pipeline to jump to the `rescue!` block

To run them you have to use `Pipeline Controller` which is part of Keechma Toolbox library, so the full example would look like this:

```
(ns pipelines.example
    (:require [keechma.toolbox.pipeline.core :as pp :refer-macros [pipeline!]
                [keechma.toolbox.pipeline.controller :as controller]))

(def controller
    (controller/constructor
        (fn [_] true)                                                          ;; this is controller's `params` function
        {:load-articles                                                        ;; pipeline key is the command it responds to
            (pipeline! [value app-db]
                (pp/commit! (assoc app-db :articles-status :loading))          ;; Mark status as `loading`
                (load-articles-from-server)                                    ;; Load some articles
                (pp/commit! (-> app-db
                              (assoc :articles-status :loaded)
                              (assoc :articles value)))                        ;; Once the articles are loaded, mark the status as `loaded` and store them in `app-db`
               (rescue! [error]
                 (pp/commit! (assoc app-db :articles-status :error))))         ;; In case of an error, mark the status as `error`

(defn register
  ([] (register {}))
  ([controllers] (assoc controllers :example controller)))
```
## ENTITYDB
Implemented as a separate library, [EntityDB](https://keechma.com/guides/entitydb/) tracks all of your application's entities. An entity is anything that is "identifiable". While the `:id` attribute is normally used for this purpose, your application can identify entities in whatever way makes sense.

Operating on a Clojure map, access to EntityDB is purely functional and supports two types of structures:

* **Collections** — lists of entities stored and retrieved by the collection's name
* **Named entity slots** — individual named entities

Since ClojureScript data structures are immutable, holding a reference to a list of entities (or an entity) will always give you the same value. By using names, EntityDB can internally update named entities and collections when the data changes and ensure data consistency.

```
(def schema {:users {:id :id})
```

Once you have schema, you can store entities:

```
(def store-v1 {})

(def user {:id 1 :username "john"})

(def store-v2 (insert-named-item schema store-v1 :users :current user))
;; store the user under the name `:current`

(get-named-item schema store-v2 :users :current)
;; Returns {:id 1 :username "john"}

(def user-collection 
 [{:id 1 :username "john" :likes "sourdough"}
  {:id 2 :username "mark" :likes "pizza"}])

(def store-v3
  (insert-collection schema store-v2 :users :list user-collection))
;; Store the user collection. User with the `:id` 1 will be updated

(get-named-item schema store-v3 :users :current)
;; Returns {:id 1 :username "retro" :likes "programming"}
```

EntityDB also supports simple relations (one-to-one and one-to-many) and more can be found at the link in title.

The nature of our apps and the rapid development time tilted the scales in favor of Dataloader (which is not a replacement for EntityDB, just a tool for loading and modeling data dependencies) so newer apps rarely take advantage of these concepts.

## DATALOADER (Keechma Toolbox)
Loading data from the back-end is a problem that every front-end app has to solve. Complex apps have to load data from multiple sources and then handle dependencies, know when to invalidate loaded data whilst keeping loading times performant. [Dataloader](https://keechma.com/news/introducing-keechma-toolbox-part-2-dataloader/) was developed as a solution those problems. Dataloader's goal is to give you the best of the component based thinking - declarative approach to data loading - while being able to easily reason about the whole app.

At its heart, Dataloader is route driven - it will automatically run on each route change. It requires you to define your data-sources and when and how they should be loaded.

Excerpt from the blog page:

> On each route change (or manual run), Dataloader will call the `:params` function for each datasource and check if the returned params are different from the previously returned value. If the value is different, Dataloader will call the `:loader` function which makes the actual request for the data, and store it wherever the `:target` attribute points to.

Example: 
```
(def restaurant-datasource
  {:target  [:edb/collection :restaurants/list]
   :loader  (map-loader
              (fn [req]
                 (when (:params req)
                  (GET "/restaurants"))))
   :params  (fn [prev {:keys [page]} deps]
                (when (= "restaurants" page) true))})
```
* `:params` function will return true only if the route's `:page` param equals to "restaurants"
* `:loader` checks if the params function returned a truthy value, and if it did it makes an AJAX request to the `/restaurants` endpoint
* returned data will be stored as an EntityDB collection named `:list`, under the `:restaurants` entity

We can also declare dependencies on datasource using `:deps` argument, we can process the resolved data using `:processor` argument, and we're not bound to some exclusive dataloaders mechanisms to load data. In a typical application, user will be require to present let's say a `JWT` token.  

```
(def ignore-datasource
  :keechma.toolbox.dataloader.core/ignore)

(def jwt-datasource
  {:target [:kv :jwt]
   :loader (map-loader #(get-item local-storage "jwt-token"))
   :params (fn [prev _ _]
             (when prev ignore-datasourrce))})
```
Interesting things to note here:
* Result is stored to `jwt` key-value pair in the app-db
* Result is loaded from the `local-storage`
* `params` function returns `ignore-datasource` - this tells the Dataloader that it shouldn't do anything with that datasource, whatever is stored in the app state is good enough

We can use that datasource later on as a dependency to some other datasource, which will tell the datasource in question not to run before this dependency is resolved.

We took this concept further by introducing `params-pipeline` which makes designing some more complex actions easier, but that toolset is not yet fully documented and I will put an example as a reference in case you encounter them:

```
(def current-restaurant
  {:target [:edb/named-item :restaurant/current]
   :deps [:jwt]
   :loader gql/loader
   :params (params-pipeline
            (guard (any
                    (select [:route :page (s/pred= "restaurant")])
                    (select [:route :page (s/pred= "favorite")]))
                   {:query [:get-restaurant-by-id :restaurant]
                    :token (ensure (select [:deps :jwt]))
                    :variables {:id (ensure (select [:route :id]))}}))})
```
Key things:

* `guard` ensures the following clauses are satisfied
* `any` (equivalent to `or`) is true when any of the following clauses is matched
* `select` - returns first element found or nil if the path does not navigate to anything or error if multiple elements found
* `ensure` - ensures element in question is present (`jwt` dependency and `:id` param from route)

 `params-pipeline` use [Specter](https://github.com/redplanetlabs/specter/) under the hood and a lot of inspiration was drawn from there.
 
 ## SUBSCRIPTIONS
 Subscriptions are functions that return a subset of the application state to the UI component. Each component declares its own dependencies. Subscriptions can be manually defined in `subscriptions.cljs` file or you can subscribe to datasources, form or route data. 
 
 Key-value pairs, or `KVs`, present the simplest form of subscriptions and look like this:
 
 ```
(ns site.subscriptions
  (:require [site.edb :as edb])
  (:require-macros [reagent.ratom :refer [reaction]]))

(defn get-kv [key]
  (fn [app-db-atom]
    (reaction
     (get-in @app-db-atom (flatten [:kv key])))))

(def subscriptions
  {:token (get-kv :token})
```
`Token` is stored in application state map under `kv` key. [Reagent's](https://reagent-project.github.io/) `reaction` is used to get the newest app-db state.

```
 (defn current-restaurant [app-db-atom]
   (reaction
    (let [app-db @app-db-atom
          restaurants (edb/get-collection app-db :restaurant :list)
          restaurant-id (get-in app-db [:route :data :id])]
      (first (filter #(= (:id %) restaurant-id) restaurant))))

(def subscriptions
  {:current-restaurant current-restaurant})
```
## COMPONENTS
Components are smaller UI entities that collectively make a system. Each one declares it's own subscription or child component dependencies. That allows Keechma to provide correct `context` to that component. If no data dependency is needed, a component in question can simply be required. Otherwise, the component needs to be configured and declared.

This is an example of `main` component.
```
(ns site.ui.main
  (:require [keechma.ui-component :as ui]
            [keechma.toolbox.ui :refer [sub> route>]]))

(defn render [ctx]
  (let [route (route> ctx)
        token (sub> ctx :token)]
   (if token
       (case (:page route)
        "dashboard" [(ui/component ctx :dashboard)]
        "loading"   [(ui/component ctx :loading)]
        [(ui/component ctx :not-found)])
       [(ui/component ctx :auth)])

(def component
 (ui/constructor
  {:renderer render
   :subscription-deps [:token]
   :component-deps [:loading
                    :dashboard
                    :not-found
                    :auth]}))
```
Couple of things to take notice of:
* Component is required with `keechma.ui-component`
* Keechma toolbox has some `ui` helpers such as `sub>` `route>` and `<cmd`
    * `sub>` gets subscription in question from components context (ctx) and dereferences it's value
    * `route>` read current route data and deref the subscription
    * `<cmd` send a command to controller
      \* _all of those actions can be accomplished manually using `ui-components` built-in functions `ui/subscription`, `ui/current-route` and `ui/send-command`_
* we use built-in `ui/component` function to render child dependency
* we setup `main` component with `ui/constructor` to which we pass configuration map in which we define `:renderer`, `:subscription-deps` and `:component-deps`

Once complete we declare component in `ui.cljs` file like this [^see NOTE]:

```
(ns site.ui
  (:require [site.ui.main :as main]
            [site.ui.loading :as loading]
            [site.ui.not-found :as not-found]
            [site.ui.dashboard :as dashboard]))
  
(def ui
 {:main       main/component   ;; app MUST contain `:main` component, this is also the key with which we declare `:component-deps`
  :loading    loading/component
  :not-found  not-found/component
  :dashboard  dashboard/component
 })  
```
> NOTE
>In newer apps we use a slightly modified declaration where we define just the map in the component file (without calling `ui/constructor`) and then pass the constructing function to all declared components in the `ui.cljs` file

One other pattern that started to emerge was the following:
Very often in our apps `main` component is used as a component-routing agent - meaning - it just reads some data and based on the result decides which child component to render. 
This is usually paired with some concept of `role` given to the end-user - `anonymous`, `viewer`, `admin` etc.

We started leveraging this mechanism throughout the app with a concept we call `rolify`. That had some impact on component naming conventions but mostly we took advantage of the namespaced keywords (a concept that gives an extra layer of depth when passing one seemingly linear information).

That resulted with a few patterns which we found to be very useful.
1. `main` component
 ```
 (ns site.ui.main
  (:require [keechma.ui-component :as ui]
            [keechma.toolbox.ui :refer [sub> route>]]
            [clojure.core.match :refer-macros [match]]))

(defn get-page [ctx]
  (let [account-role (sub> ctx :account-role)
        route        (route> ctx)]
    (match [account-role route]
           [_ {:page "loading"}]           :loading

           [:anon {:page "login"} ]        :anon/login
           [:anon {:page "homepage"} ]     :anon/login

           [:admin {:page "homepage"}]     :shared/homepage
           [:admin {:page "users"}]        :admin/users
           [:admin {:page "users" :id _}]  :admin/user

           [:editor {:page "homepage"}]    :shared/homepage
           [:editor {:page "tasks"}]       :shared/tasks
           [:editor {:page "tasks" :id _}] :editor/task

           [:viewer {:page "homepage"}]    :shared/homepage
           [:viewer {:page "tasks"}]       :shared/tasks

      :else :not-found)))

(defn render [ctx]
  (let [page      (get-page ctx)
        component (ui/component ctx page)]
    [component]))

(def component
  {:renderer          render
   :subscription-deps [:account-role]
   :component-deps    [:loading
                       :not-found
                       :anon/login
                       :shared/homepage
                       :shared/tasks
                       :admin/users
                       :admin/user
                       :editor/task]})
```
 * we pattern match `route` and `account-role` data here to redirect user to correct page
 * we can also use shared components where needed
 * we can easily handle `404`, `loading` or `unauthorized` pages

2. **Forms**
    \- An added layer of security (role-based forms, mounted only if valid role present)
3. **Controllers**
    \- Role-based action validations
4. **UI**
    \- Role-based UI behavior
5. **Route processing**
    \- Role-based route processing

## FORMS (SHORT)
TBD

## FOREIGN LIBS
TBD

## PROJECT STRUCTURE
Project structure stays mostly the same from project to project, but as the framework evolves and new tools come to light small changes can be expected. 
The core structure follows the [Figwheels template](https://github.com/bhauman/lein-figwheel) for [Leiningen project](http://alexott.net/en/clojure/ClojureLein.html) and the main parts of our code can be found in `src/cljs` subfolder which will focus on.
```
example-project
|   README.md
|   project.clj
|   ...
|   
└───src
    └───cljs
        │   controllers.cljs                                     ;; place to register controllers
        │   core.cljs                                            ;; where core configuration takes place
        │   datasources.cljs                                     ;; declaration and registration of datasources (optional, see [1])
        │   edb.cljs                                             ;; EntityDB schema definitions go here [2]
        │   forms.cljs                                           ;; place where forms are registered and their mount rules are declared [3]
        │   rn.cljs                                              ;; file to declare and adapt external (react) libraries
        │   stylesheet.cljs                                      ;; core styles and style helpers
        │   subscriptions.cljs                                   ;; create and register subscriptions [4]
        │   ui.cljs                                              ;; place to register components and pages [5]
        │   util.cljs                                            ;; general utilities file
        │   ...
        │ 
        └───controllers                                          ;; controllers  
        │       ...
        └───domain                                               ;; domain stuff 
        │       ...
        └───forms                                                ;; forms logic and setup
        │       ...
        └───ui                                                   ;; pages and components
            │   main.cljs                                        ;; UI entry file
            │   ...
            │
            └───components                                       ;; place for reusable components and 
            │   |    inputs.cljs
            │   |    ...
            │   |    
            │   └───pure 
            │        ...
            └───pages
                |    ...
                └───page_name

```
[1] - [Datasources](https://keechma.com/news/introducing-keechma-toolbox-part-2-dataloader/) - part of Keechma's Toolbox; data loading and data management is a question each front-end app must solve and Dataloader is Keechma's answer to that; a way of declaring data, it's relations, dependencies and invalidation rules at once without a need to hold the complex mental model that glues everything together.
[2] - [EntityDB](https://keechma.com/guides/entitydb/) - EntityDB represents Keechma's data storage system; it provides tools for keeping data consistent without need for manual synchronization of some kind; Datasources are a normalized and structured way of loading and storing data, so using one in favor of other is customary but not exclusive.
[3] - [Forms](https://keechma.com/news/announcing-keechma-forms/) - part of Keechma's ecosystem, meant to make form validations and state management more manageable
[4] - Subscriptions - functions that return a subset of the application state to UI components; each declared Datasource is also a subscription of its own
[5] - [UI Layer](https://keechma.com/guides/ui-system/) - non-pure pages and components (one that display reactive data) must be registered in `ui.cljs` file; `main` component is a must

In the `role` based concept `UI` layer would be further separated into corresponding subdirectories:
```
└───ui  
    │   main.cljs 
    │   ...
    │
    └───anon                                     
    │    foo.cljs
    │   ...
    └───admin                                  
    │    foo.cljs
    │   ...
    └───shared                               
    │    foo.cljs
    │   ...
```

## HTML/CSS MOST USED CONCEPTS AND HELPERS
### 1. `ui/url` fn
- most commonly used for generating `href` links
- returns a URL based on the params. It will use the :url-fn that is injected from the outside to generate the URL based on the current app routes
- for example if the route patterns were setup to `[":page/:id"]`
`[:a {:href (ui/url ctx {:page "user" :id 1})} "Link"]` would produce an URL of `/user/1`
### 2. `ui/redirect`
- redirects page to the URL generated from params
- most commonly used to redirect from `click` actions
- example
`[:button {:on-click #(ui/redirect ctx {:page "user" :id 1})} ...]`
### 3. `<cmd`
- used to send command to a particular controller 
- example:
`[:button {:on-click #(<cmd ctx [:actions :do-something] :foo} ...]`
- this would send a command `:do-something` to the controller `:actions` with the payload of `:foo`
### 4. `class-names` 
- from `[keechma.toolbox.util :refer [class-names]]`
- applies class-name when `condition` is true
- example:
```
[:div {:class (class-names {:active (= "foo" user-favorite)})]
```
### 5. `Defelement` macro
- `defelement` is a way of declaring an element and defining it's styling properties to be used in code. This gives us a couple of benefits:
1. It decouples code styling from code functionality and layout which gives as an easier mental model to work with and clearer lay of the land when encountering complex components
2. Declared elements can be reused across the app 
3. If properly used they make future changes easier and faster. Ie, if we have one `styled-heading` in the app and the styling parameters change, we have to update code in one place only.
4. Defelements provide support for rapid element styling using helper classes and additional support for any custom style needs.

### Structure
```
(defelement -header                                         ;; Name, prefixed with a hyphen to differentiate from other elements in hiccup
  :tag :div                                                 ;; html element to create, if ommited defaults to `div`
  :class [:flex :bg-dark-blue :py4 :mb4 :justify-between]   ;; class helpers used to style the entity
  :style [{:width "120px"}])                                ;; custom style goes here
```
- class helpers used to style element from various sources; basic CSS toolkit used here is https://basscss.com/ and any class helper stated there can be used here
- additional classes are sourced from the apps `stylesheet.cljs`/`styles.cljs` where various helpers are generated or staticly declared classes are defined
- examples of oftenly used in-house classes are:
-- `bg-color_name` and `c-color_name` are used to set css properties of `color` and `background-color`, `color_name` can be declared in `stylesheet/colors.cljs` file:

```
(def colors 
 {:white        "#ffffff"
  :black        "#000000"
  :gray    "#525263"
  ...}
```
 Botth color classes and helper classes will be generated for each one, ie `bg-gray`; this way colors are declared in one place and can easily be update if needed
-- various modifiers are also created alongside basic classes:
--- `c-h-black` which triggers `color` property on `:hover`
--- `c-gray-d` or `c-gray-l` which provides slightly darker or lighter variant of stated color
--- `t-c` or `t-bg` which applies predefined `transition` properties to `c` color and `bg` background

- coupled together we can easily produce
```
(defelement -styled-link
 :tag :a
 :class [:bg-gray-l :bg-h-gray :bg-t :c-blue :c-h-blue-d :c-t])
```
which will created an `anchor` element with:
 - `bg-gray-l`   - background-color of lighter variant of `gray`
 - `bg-h-gray`   - that transitions to `gray` on `:hover`
 - `bg-t`        - and has predefined `transitions` style applied
 - `c-blue`      - while having text color of `blue`
 - `c-h-blue-d`  - which transitions to darker `blue` on `:hover`
 - `c-t`         - with `transitions` style applied

Coupled together the end code will look something like:
```
(defelement -header                                         
  :tag :div                                                 
  :class [:flex :bg-dark-blue :py4 :mb4 :justify-between]   
  :style [{:width "120px"}])
  
(defelement -styled-link
 :tag :a
 :class [:bg-gray-l :bg-h-gray :bg-t :c-blue :c-h-blue-d :c-t])

(defn render [] 
 [:div                                ;; regular <div> for example purposes
  [-header {:src "/image/foo.jpg"}]
  [-styled-link "Some link"]])
 ```
 Together with proper functionality separation this produces a component thats easier to "read" - wrap your head around its components style, layout and functionality - and maintain in the long run.