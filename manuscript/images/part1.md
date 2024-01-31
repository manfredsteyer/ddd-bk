# Modern Architectures with Angular - Part 1: Strategic Design with Sheriff and Standalone Components

<iframe width="560" height="315" src="https://www.youtube.com/embed/Y1_KwcJxQKg" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

Angular is often used for the frontend of large, mission-critical solutions. In this environment in particular, it is particularly important to ensure that the architecture is easy to maintain. However, it is also important to avoid over-engineering. Current features such as standalone components or standalone APIs help.

In this two-part series of articles, I show how you can reconcile both requirements. The first part highlights the implementation of your strategic design based on standalone components and standalone APIs. The specified architecture is enforced with the open-source Sheriff project.

The examples used here work with a traditional Angular CLI-Project but also with Nx. In the latter case, we recommend starting with an Nx Standalone-Application to keep the architecture lightweight and folder-based.

[Source Code](https://github.com/manfredsteyer/modern-arc.git)

## The Guiding Theory: Strategic Design from DDD

Strategic Design, one of the two original disciplines of Domain-driven Design (DDD), has proven itself as the guiding theory for structuring modern front ends. The main point here is to break down a software system into various technical sub-domains. For example, in an airline, you might find the following sub-domains:

<img alt="Domain Cut" src="https://www.angulararchitects.io/wp-content/uploads/2023/07/domains.png" style="max-width:400px !important">

Identifying individual domains requires a look at the business processes to be supported. The interaction between developers and architects on the one hand and domain experts on the other is essential. Workshop formats such as event storming [Event Storming](https://www.eventstorming.com/), which combine DDD with ideas from agile software development, are ideal for this.

A context map represents the relationships and dependencies between individual domains:

<img alt="A simple context map" src="https://www.angulararchitects.io/wp-content/uploads/2023/07/context-map.png" style="max-width:400px !important">

The goal is to decouple the individual domains from each other: the less they know about each other, the better it is. This prevents changes to one applied part from affecting others, thus improving maintainability. For larger projects, it is common to assign one or more domains to each sub-team.

In order to achieve the desired decoupling, _Booking could_ only publish a few selected services in the example shown. Alternatively, information about booked flights could be distributed via messaging in the backend. In addition, Strategic Design offers numerous other patterns and considerations that help to establish a loose coupling.

## Domains vs Bounded Context

Strictly speaking, a domain is implemented as one or more bounded contexts, and each bounded context can also contain one or more domains. Accordingly, the bounded context reflects the solution view, while the term domain represents part of the problem view.

Each bounded context has a domain model that reflects the respective functionalities, e.g. the structure and handling of flights and tickets. This domain model only makes sense within the bounded context. Even if the same terms appear in other contexts, those contexts most likely have a different view of it. From booking's point of view, a flight is designed differently than from the boarding's one. These two views are deliberately implemented separately. This prevents both coupling between contexts and the creation of a confusing model that tries to describe too much at once.

For the sake of simplicity, the implementation in this article assumes that there is one bounded context per domain.


## Transition to Source Code: The Architecture Matrix 

For mapping in the source code, it makes sense to further subdivide the individual domains into different modules:

![Matrix](https://www.angulararchitects.io/wp-content/uploads/2023/07/matrix.png)

A categorization of these modules increases clarity. [Nrwl](https://go.nrwl.io/angular-enterprise-monorepo-patterns-new-book) suggests the following categories (originally for libraries), among others, which have proven helpful in our daily work:

- **feature:** A feature module implements a use case (or a technical feature) with so-called smart components. Due to their focus on a feature, such components are not very reusable. Smart Components communicate with the backend. Typically, in Angular, this communication occurs through a store or services.
- **ui:** UI modules contain so-called dumb or presentational components. These are reusable components that support the implementation of individual features but do not know them directly. The implementation of a design system consists of such components. However, UI modules can also contain general technical components that are used across all use cases. An example of this would be a ticket component, which ensures that tickets are presented in the same way in different features. Such components usually only communicate with their environment via properties and events. They do not get access to the backend or a store.
- **data:** Data modules contain the respective domain model (actually the client-side view of it) and services that operate on it. Such services validate e.g. Entities and communicating with the backend. State management, including the provision of view models, can also be accommodated in data modules. This is particularly useful when multiple features in the same domain are based on the same data.
- **util:** General helper functions etc. can be found in utility modules. Examples of this are logging, authentication or working with date values.

Another special aspect of the implementation in the code is the shared area, which offers code for all domains. This should primarily have technical code – use case-specific code is usually located in the individual domains.

The structure shown here brings order to the system: There is less discussion about where to find or place certain sections of code. In addition, two simple but effective rules can be introduced on the basis of this matrix:

- In terms of strategic design, each domain may only communicate with its own modules. An exception is the shared area to which each domain has access.
- Each module may only access modules in lower layers of the matrix. Each module category becomes a layer in this sense.

Both rules support the decoupling of the individual modules or domains and help to avoid cycles.

## Project Structure for the Architecture Matrix

The architecture matrix can be mapped in the source code in the form of folders: Each domain has its own folder, which in turn has a subfolder for each of its modules:

<img alt="Folder structure with domains" src="https://www.angulararchitects.io/wp-content/uploads/2023/07/folder-structure-02.png" style="max-width:250px !important">

The module names are prefixed with the name of the respective module category. This means that you can see at first glance where the respective module is located in the architecture matrix. Within the modules are typical Angular building blocks such as components, directives, pipes, or services.

The use of Angular modules is no longer necessary since the introduction of standalone components (directives and pipes). Instead, the _standalone_ flag is set to _true:_

```typescript
@Component({
  selector: 'app-flight-booking',
  standalone: true,
  imports: [CommonModule, RouterLink, RouterOutlet],
  templateUrl: './flight-booking.component.html',
  styleUrls: ['./flight-booking.component.css'],
})
export class FlightBookingComponent {
}
```

In the case of components, the so-called compilation context must also be imported. These are all other standalone components, directives and pipes that are used in the template.

An _index.ts_ is used to define the module's public interface. This is a so-called barrel that determines which module components may also be used outside of the module:

```typescript
export * from './flight-booking.routes';
```

Care should be taken in maintaining the published constructs, as breaking changes tend to affect other modules. Everything that is not published here, however, is an implementation detail of the module. Changes to these parts are, therefore, less critical.

## Enforcing your Architecture with Sheriff

The architecture discussed so far is based on several conventions:

- Modules may only communicate with modules of the same domain and _shared_
- Modules may only communicate with modules on lower layers
- Modules may only access the public interface of other modules

The Sheriff [Sheriff](https://github.com/softarc-consulting/sheriff) open-source project allows these conventions to be enforced via linting. Violation is warned with an error message in the IDE or on the console:

![Sheriff in the IDE](https://www.angulararchitects.io/wp-content/uploads/2023/07/sheriff.png)

![Sheriff in the console](https://www.angulararchitects.io/wp-content/uploads/2023/07/linter.png)

The former provides instant feedback during development, while the latter can be automated in the build process. This can be used to prevent, for example, source code that violates the defined architecture from ending up in the _main_ or _dev_ branch of the source code repo.

To set up Sheriff, the following two packages must be obtained via npm:

```javascript
npm i @softarc/sheriff-core @softarc/eslint-plugin-sheriff -D
```

The former includes Sheriff himself, the latter is the tethering to _eslint_ . The latter must be registered in the _.eslintrc.json_ in the project root:

```json
{
  [...],
  "overrides": [
    [...]
    {
      "files": ["*.ts"],
      "extends": ["plugin:@softarc/sheriff/default"]
    }
  ]
}
```

Sheriff considers any folder with an _index.ts_ as a module. By default, Sheriff prevents this _index.js_ from being bypassed and thus access to implementation details by other modules. The _sheriff.config.ts_ to be set up in the root of the project defines categories (_tags_) for the individual modules and defines dependency rules (_depRules_) based on them. The following shows a Sheriff configuration for the architecture matrix discussed above:

```typescript
import { noDependencies, sameTag, SheriffConfig } from '@softarc/sheriff-core';

export const sheriffConfig: SheriffConfig = {
  version: 1,

  tagging: {
    'src/app': {
      'domains/<domain>': {
        'feature-<feature>': ['domain:<domain>', 'type:feature'],
        'ui-<ui>': ['domain:<domain>', 'type:ui'],
        'data': ['domain:<domain>', 'type:data'],
        'util-<ui>': ['domain:<domain>', 'type:util'],
      },
    },
  },
  depRules: {
    root: ['*'],

    'domain:*': [sameTag, 'domain:shared'],

    'type:feature': ['type:ui', 'type:data', 'type:util'],
    'type:ui': ['type:data', 'type:util'],
    'type:data': ['type:util'],
    'type:util': noDependencies,
  },
};
```

The tags refer to folder names. Expressions such as _\<domain\>_ or _\<feature\>_ are placeholders. Each module below _src/app/domains/\<domain\>_ whose folder name begins with _feature-_ is therefore assigned the categories _domain:\<domain\>_ and _type:feature_ . In the case of _src/app/domains/booking,_ these would be the categories _domain:booking_ and _type:feature_ .

The dependency rules under _depRules_ pick up the individual categories and stipulate, for example, that a module only has access to modules in the same domain and to _domain:shared ._ Further rules define that each layer only has access to the layers below it. Thanks to the _root: ['\*']_ rule, all non-explicitly categorized folders in the root folder and below are allowed access to all modules. This primarily affects the shell of the application.

## Lightweight Path Mappings

Path mappings can be used to avoid illegible relative paths within the imports. These allow, for example, instead of

```typescript
import { FlightBookingFacade } from '../../data';
```

to use the following:

```typescript
import { FlightBookingFacade } from'@demo/ticketing/data' ;
```

Such three-character imports consist of the project name or name of the workspace (e.g. _@demo_), the domain name (e.g. _ticketing_), and a module name (e.g. _data_) and thus reflect the desired position within the architecture matrix.

This notation can be enabled independently of the number of domains and modules with a single path mapping within _tsconfig.json_ in the project root:

```json
{
  "compileOnSave": false,
  "compilerOptions": {
    "baseUrl": "./",
    [...]
    "paths": {
      "@demo/*": ["src/app/domains/*"],
    }
  },
  [...]
}
```

IDEs like Visual Studio Code should be restarted after this change. This ensures that they take this change into account.

## Standalone APIs

Since standalone components make the controversial Angular modules optional, the Angular team now provides so-called standalone APIs for registering libraries. Known examples are _provideHttpClient_ and _provideRouter:_

```typescript
bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient(),
    provideRouter(APP_ROUTES, withPreloading(PreloadAllModules)),

    importProvidersFrom(NextFlightsModule),
    importProvidersFrom(MatDialogModule),

    provideLogger({
        level: LogLevel.DEBUG,
    }),
  ],
});
```

Essentially, these are functions that providers return for the required services. The choice of this provider and, thus, the behavior of the library can be influenced by passing a configuration object. An example of this is the route configuration that _provideRouter_ accepts.

From an architectural point of view, standalone APIs serve another purpose: They allow a system component to be viewed as a black box that can be further developed independently. The black box can also become a gray box by transferring a configuration object. In this case, the behavior of the system component used can be adjusted via well-defined settings without having to give up the loose coupling. The open/closed principle is also used here: Open for extensions (through configuration or polymorphism), closed for modifications by the consumer.

As an example of a custom standalone API that sets up a logger, the former listing called _provideLogger_:

```typescript
export function provideLogger(
  config: Partial<LoggerConfig>
): EnvironmentProviders {
  const merged = { ...defaultConfig, ...config };

  return makeEnvironmentProviders([
    LoggerService,
    {
      provide: LoggerConfig,
      useValue: merged,
    },
    {
      provide: LOG_FORMATTER,
      useValue: merged.formatter,
    },
    merged.appenders.map((a) => ({
      provide: LOG_APPENDERS,
      useClass: a,
      multi: true,
    })),
  ]);
}
```

The _provideLogger function_ takes a partial _LoggerConfig_ object. The caller, therefore, only has to deal with those settings that are relevant for the current case. In order to get a complete _LoggerConfig, provideLogger_ merges the given configuration with a default configuration. Based on this, it returns various providers.

The _makeEnvironmentProviders_ function from _@angular/core_ wraps the constructed provider array with an object of type _EnvironmentProviders_. This type can be used when bootstrapping the application and within routing configurations. It thus allows providers to be provided for the entire application or individual parts.

Unlike a traditional provider array, _EnvironmentProviders_ cannot be used inside components. This limitation is intentional since most libraries, such as the router, are designed for cross-component use.

## What's next? More on Architecture!

Please find more information on enterprise-scale Angular architectures in our free eBook (5th edition, 12 chapters): 

- According to which criteria can we subdivide a huge application into sub-domains?
- How can we make sure, the solution is maintainable for years or even decades?
- Which options from Micro Frontends are provided by Module Federation?

<a href="https://www.angulararchitects.io/book"><img src="https://www.angulararchitects.io/wp-content/uploads/2022/06/cover-5th-small.png" alt="free ebook"></a>

Feel free to [download it here](https://www.angulararchitects.io/en/book) now!


## Conclusion

Strategic design subdivides a system into different ones that are implemented as independently as possible. This decoupling prevents changes in one area of application from affecting others. The architecture approach shown subdivides the individual domains into different modules, and the open-source project Sheriff ensures that the individual modules only communicate with one another in respecting the established rules.

This approach allows the implementation of large and long-term maintainable frontend monoliths. Due to their modular structure, the language is sometimes also of moduliths. A disadvantage of such architectures is increased build and test times. This problem can be solved with incremental builds and tests. The second part of this series of articles addresses this.
