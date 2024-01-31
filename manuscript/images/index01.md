# Micro Frontends with Modern Angular - Part 1: Standalone and esbuild

Angular is changing. Standalone Components make the framework more lightweight and providing Web Components via Angular Elements easier. The new esbuild integration, which will be on by default beginning with version 17, offers a significant improvement of build performance over the more traditional webpack-based build process. In our tests, we were able to speed up our build processes by a factor of 3 to 4. Angular 17 also brings revised support for server-side rendering. Together with the current efforts regarding hydration and the planned deferred loading, this can drastically improve runtime performance, among other things.

In this article series, I'm looking into all these topics. In this part, I'm starting with Standalone Components and esbuild.

[Source Code](https://github.com/manfredsteyer/standalone-example-cli) 
(see branches `nf-standalone-solution` and `nf-standalone-router-config`)

## Module Federation as a Game Changer

Module Federation -- shipped with webpack since version 5 -- is often seen as a game changer for micro frontends. It allows separately compiled and published application components to be loaded on demand:

![Basic functionality of Module Federation](https://www.angulararchitects.io/wp-content/uploads/2023/10/schema.png)

A shell application (officially called host) defines Url segments that point to the Micro Frontends (officially called remotes). The Micro Frontends publish program parts, such as components or Angular modules. The shell can now load these program parts at runtime.

Furthermore, Module Federation also allows to share dependencies at runtime. Angular only needs to be loaded once, even if several separately compiled and published Micro Frontends are based on it.

## Angular CLI: esbuild replaces webpack and thus Module Federation!

Apart from a few pre-release versions, the Angular CLI has been using webpack for the build since its early days. For good reason, because for a long time webpack was the de facto standard in this area. However, there are now some alternatives that offer significantly better build performance. This is made possible by using native languages instead of JavaScript and supporting parallelization from the ground up.

The most popular of these alternatives is `esbuild` [esbuild]. Since Angular 16, the CLI comes with a developer preview of an `esbuild` builder. From version 17 this should be activated by default. As a result, both `ng serve` and `ng build` run noticeably faster.

However, switching to `esbuild` brings a challenge for Micro Frontends: The popular Module Federation comes with webpack and is not available for `esbuild`. The next sections provide a solution.

## Native Federation with esbuild

In order to be able to use the proven mental model of Module Federation independently of webpack, the [Native Federation](https://www.npmjs.com/package/@angular-architects/native-federation) project was created. It offers the same options and configuration as Module Federation, but works with all possible build tools. It also uses browser-native technologies such as EcmaScript modules and [Import Maps](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/script/type/importmap). This measure is intended to ensure long-term support from browsers and also allow alternative implementations.

Native Federation is called before and after the actual bundler in the build process. That's why it doesn't matter which bundler is actually used:

![Native Federation extends existing build scripts](https://www.angulararchitects.io/wp-content/uploads/2023/10/native-federation.png)

Since Native Federation also needs to create a few bundles, it delegates to the bundler of choice. The individual bundlers are connected via interchangeable adapters.

The following image shows an example built with Angular, esbuild, and Native Federation:

![Shell with separately deployed micro frontend](https://www.angulararchitects.io/wp-content/uploads/2023/10/example.png)

The shell shown here has loaded a separately developed and deployed Micro Frontend into its workspace using Native Federation.

Although both the shell and the micro frontend are based on Angular, Native Federation **only loaded Angular once.** To make this possible, Native Federation, following the ideas of Module Federation, places the remotes and the libraries to be shared in their own bundles. For this, it uses standards-compliant EcmaScript bundles that could also be created by other tools. Information about these bundles is placed in metadata files:

![Native Federation at runtime](https://www.angulararchitects.io/wp-content/uploads/2023/10/native-federation-runtime-1.png)

These metadata files are the basis for a standard-compliant Import Map that informs the browser from where which bundles are to be loaded.

## Native Federation: Setting up a Micro Frontend 

For use with Angular and the CLI, Native Federation offers an `ng-add` schematic. The following statement adds Native Federation to the Angular project `mfe1` and configures it as a `remote` acting as a Micro Frontend:

```bash
ng add @angular-architects/native-federation --project mfe1 --port 4201 --type remote
```

The `ng-add`-Schematic also creates a `federation.config.js` controlling Native Federation's behavior:

```javascript
const { withNativeFederation, shareAll } = require('@angular-architects/native-federation/config');

module.exports = withNativeFederation({

  name: 'mfe1',

  exposes: {
    './Component': './projects/mfe1/src/app/app.component.ts',
  },

  shared: {
    ...shareAll({ singleton: true, strictVersion: true, requiredVersion: 'auto' }),
  },

  skip: [
    'rxjs/ajax',
    'rxjs/fetch',
    'rxjs/testing',
    'rxjs/webSocket',
    // Add further packages you don't need at runtime
  ]
  
});
```

The property `name` defines a unique name for the remote. The `exposes` section specifies which files the remote should expose to the host. Although these files are built and deployed together with the remote, they can be loaded into the host at runtime. Since the host doesn't care about the full file paths, `exposes` maps them to shorter names.

In the case shown, the remote just publishes its `AppComponent` for simplicity. However, any system component could be published instead, e.g. lazy routing configurations that reference multiple components of a feature.

Under `shared`, the configuration lists all dependencies that the remote wants to share with other remotes and the host. In order to avoid an exhaustive list of all required npm packages, the `shareAll` helper function is used. It includes all packages that are in the `package.json` under `dependencies`. Details about the parameters passed to _shareAll_ can be found in [this blog article](https://www.angulararchitects.io/en/blog/getting-out-of-version-mismatch-hell-with-module-federation/).

Packages `shareAll` should not share are entered under `skip`. This can improve the build and startup performance of the application slightly. In addition, packages that are intended for use with **NodeJS must be added to `skip`**, since they cannot be compiled for use in the browser.

## Native Federation: Setting up a Shell

The host acting as a Micro Frontend Shell can also be set up with `ng add`:

```bash
ng add @angular-architects/native-federation --project shell --port 4200 --type dynamic-host
```

The type `dynamic-host` indicates that the remotes to be loaded are defined in a configuration file:

```json
{
    "mfe1" : "http://localhost:4201/remoteEntry.json"
}
```

This `federation.manifest.json` is generated in the host's `assets` folder by default. By treating it as an asset, the manifest can be exchanged during deployment. The application can therefore be adapted to the respective environment.

The manifest maps the names of the remotes to their metadata, which Native Federation places in the `remoteEntry.json` file during build. Even if `ng add` generates the manifest, it should be checked in order to adjust ports if necessary or to remove applications that are not remotes.

The `ng add`-command also generates a `federation.config.js` for hosts:

```javascript
const { withNativeFederation, shareAll } = require('@angular-architects/native-federation/config');

module.exports = withNativeFederation({

  shared: {
    ...shareAll({ singleton: true, strictVersion: true, requiredVersion: 'auto' }),
  },

  skip: [
    'rxjs/ajax',
    'rxjs/fetch',
    'rxjs/testing',
    'rxjs/webSocket',
    // Add further packages you don't need at runtime
  ]
  
});
```

The `exposes` entry known from the remote's config is not generated for hosts because hosts typically do not publish files for other hosts. However, if you want to set up a host that also acts as a remote for other hosts, there is nothing wrong with adding this entry.

The `main.ts` file, also modified by `ng add`, initializes Native Federation using the manifest:

```typescript
import { initFederation } from '@angular-architects/native-federation';

initFederation('/assets/federation.manifest.json')
  .catch(err => console.error(err))
  .then(_ => import('./bootstrap'))
  .catch(err => console.error(err));
```

The `initFederation` function reads the metadata of each remote and generates an Import Map used by the browser to load shared packages and exposed modules. The program flow then delegates to the `bootstrap.ts`, which starts the Angular solution with the usual instructions (`bootstrapApplication` or `bootstrapModule`).

All files considered so far were set up using `ng add`. In order to load a program part published by a remote, the host must be expanded to include lazy loading:

```typescript
[…]
import { loadRemoteModule } from '@angular-architects/native-federation';

export const APP_ROUTES: Routes = [
  […],
  {
    path: 'flights',
    loadComponent: () =>
      loadRemoteModule('mfe1', './Component').then((m) => m.AppComponent),
  },
  […]
];
```

The lazy route uses the `loadRemoteModule` helper function to load the `AppComponent` from the remote. It takes the name of the remote from the manifest (`mfe1`) and the name under which the remote publishes the desired file (`./Component`).


## Exposing a Router Config

Just exposing one component via Native Federation is a bit fine-grained. Quite often, you want to expose a whole feature that consists of several components. Fortunately, we can expose all kinds of TypeScript/EcmaScript constructs. In the case of coarse-grained features, we could expose an NgModule with subroutes or -- if we go with Standalone Components -- just a routing config. Here, the latter one is the case:

```typescript
import { Routes } from "@angular/router";
import { FlightComponent } from "./flight/flight.component";
import { HolidayPackagesComponent } from "./holiday-packages/holiday-packages.component";

export const APP_ROUTES: Routes = [
    {
        path: '',
        redirectTo: 'flights',
        pathMatch: 'full'
    },
    {
        path: 'flight-search',
        component: FlightComponent
    },
    {
        path: 'holiday-packages',
        component: HolidayPackagesComponent
    }
];
```

This routing config needs to be added to the `exposes` section in the Micro Frontend's `federation.config.js`:

```typescript
const { withNativeFederation, shareAll } = require('@angular-architects/native-federation/config');

module.exports = withNativeFederation({

  name: 'mfe1',

  exposes: {
    './Component': './projects/mfe1/src/app/app.component.ts',

     // Add this line:
    './routes': '././projects/mfe1/src/app/app.routes.ts',
  },

  shared: {
    ...shareAll({ singleton: true, strictVersion: true, requiredVersion: 'auto' }),
  },

  skip: [
    'rxjs/ajax',
    'rxjs/fetch',
    'rxjs/testing',
    'rxjs/webSocket',
    // Add further packages you don't need at runtime
  ]
  
});
```

In the shell, you can directly route to this routing configuration:

```typescript
[...]
import { loadRemoteModule } from '@angular-architects/native-federation';

export const APP_ROUTES: Routes = [
  [...]

  {
    path: 'flights',
    // loadChildreas instead of loadComponent !!!
    loadChildren: () =>
      loadRemoteModule('mfe1', './routes').then((m) => m.APP_ROUTES),
  },

  [...]
];
```

Also, we need to adjust the routes in the shell's navigation:

```html
<ul>
    <li><img src="../assets/angular.png" width="50"></li>
    <li><a routerLink="/">Home</a></li>
    <li><a routerLink="/flights/flight-search">Flights</a></li>
    <li><a routerLink="/flights/holiday-packages">Holidays</a></li>
</ul>

<router-outlet></router-outlet>
```

## Communication between Micro Frontends

Communication between Micro Frontends can also be enabled via shared libraries. I'd like to say in advance that this option should only be used with caution: Micro Frontend architectures are intended to decouple individual frontends from each other. However, if a frontend expects information from other frontends, exactly the opposite is achieved. Most solutions I've seen only share a handful of contextual information. Examples include the current username, the current client and a few global filters.

To share information, we first need a shared library. This library can be a separately developed npm package or a library within the current Angular project. The latter can be generated with

```bash
ng g lib auth
```

The name of the library in this case is set as `auth`. To share data, this library receives a stateful service. For the sake of brevity, I'm using the simplest stateful service I can think about:

```typescript
@Injectable({
  providedIn: 'root'
})
export class AuthService {
  userName = '';
}
```

In this very simple scenario, the service is used as a black board: A Micro Frontend writes information into the service and another one reads the information. However, a somewhat more convenient way to share information would be to use a publish/subscribe mechanism through which interested parties can be informed about value changes. This idea can be realized, for example, by using RxJS subjects.

If Monorepo-internal libraries are used, they should be made accessible via path mapping in the `tsconfig.json`:

```json
"compilerOptions": {
    "paths": {
      "@demo/auth": [
        "projects/auth/src/public-api.ts"
      ]
     },
     […]
}
```

Please note that I'm pointing to `public-api.ts` in the **lib's source code.** This strategy is also used by Nx, but the CLI points to the `dist` folder by default. Hence, in the latter case, you need to update this entry by hand.

It must also be ensured that all communication partners use the same path mapping. 

## Conclusion

The new esbuild builder provides tremendous improvements in build performance. However, the popular Module Federation is currently bound to webpack. Native Federation provides the same mental model and is implemented in a tooling-agnostic way. Hence, it works with all possible bundlers. Also, it uses web standards like EcmaScript modules and Import Maps. This also allows for different implementation and makes it a reliable solution in the long run. 


#  Micro Frontends with Modern Angular - Part 2: Multi-Version and Multi-Framework Solutions with Angular Elements and Web Components

The first part of this article series showed how to use modern Angular with Native Federation and esbuild. We've assumed that all Micro Frontends and the shell use the same framework in the same version. However, if the goal is to integrate Micro Frontends that are based on different frameworks and/or framework versions, you need some additional considerations.

As [discussed in this blog article](https://www.angulararchitects.io/blog/multi-framework-and-version-micro-frontends-with-module-federation-your-4-steps-guide/), such an approach is nothing you normally want to introduce without a good reason like dealing with legacy systems or combining existing products to a suite.

[Source Code](https://github.com/manfredsteyer/standalone-example-cli) 
(see branch `nf-web-comp-mixed`)

## Abstracting Micro Frontends with Web Components

The first step when implementing such a scenario is abstracting the different frameworks and versions. A popular way for this is to use Web Components that encapsulate entire Micro Frontends. The result is not a ideal-typical Web Components in the sense of reusable widgets, but rather a coarse-grained web components that represent entire domains. The following image shows a Web Component bootstrapping a React application that is loaded into an Angular-based shell:

![React-Micro frontend in Angular Shell](https://www.angulararchitects.io/wp-content/uploads/2023/10/multi.png)

Actually, it's not difficult to write a Web Component that delegates to a framework instead of writing something into the DOM itself. Angular makes even this task easier with the `@angular/elements` converting an Angular component into a Web Component. Technically speaking, it wraps the Angular Component into a Web Component created on the fly.

The package `@angular/elements` can be installed with npm (`npm i @angular/elements`). Together with Standalone Components, it's used in a straight forward manner:

```typescript
import { NgZone } from '@angular/core';

(async () => {
  const app = await createApplication({
    providers: […],
  });

  const mfe2Root = createCustomElement(AppComponent, {
    injector: app.injector,
  });

  customElements.define('mfe2-root', mfe2Root);
})();
```

The lines of code shown here replace bootstrapping the application. The `createApplication` function creates an Angular application including a root injector. The latter is configured via the _providers_ array. However, instead of bootstrapping a component, `createCustomElement` transforms a standalone component into a web component.

The `customElements.define` method is already part of the browser's API and registers the Web Component under the name `mfe2-root`. This means that from now on the browser will render the web component and thus the Angular component behind it as soon as `<mfe2-root></mfe2-root>` appears in the markup. When assigning the name, please note that by definition it must contain a hyphen. This is to avoid naming conflicts with existing HTML elements.

In order to share this Web Component via Native Federation, the file defining the Web Component must be specified in the `federation.config.js` under `exposes`. In the case considered here, this is `bootstrap.ts`:

```json
exposes: {
  './web-components': './projects/mfe2/src/bootstrap.ts',
},
```

This approach gives us the best of both worlds: Using Native Federation, frameworks and libraries can be shared as long as multiple Micro Frontends use the same version. By abstracting differences via Web Components, different frameworks and framework versions can also be integrated:

![Web Component and Native Federation: The best of two worlds](https://www.angulararchitects.io/wp-content/uploads/2023/10/venn.png)

## Loading Web Components in a Shell

Providing a Web Component via Native Federation is only one side of the coin: We also have to load the Web Component into a shell. Since the Angular Router can only work with Angular Components, it makes sense to wrap our Web Components into an Angular Component:

```typescript
import { loadRemoteModule } from '@softarc/native-federation-runtime';

@Component({
  selector: 'app-wrapper',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './wrapper.component.html',
  styleUrls: ['./wrapper.component.css']
})
export class WrapperComponent implements OnInit {
  elm = inject(ElementRef);

  async ngOnInit() {
    await loadRemoteModule('mfe2', './web-components');
    const root = document.createElement('mfe2-root');
    this.elm.nativeElement.appendChild(root);
  }
}
```

This `WrapperComponent` loads the Web Component via Native Federation and creates an HTML element into which the browser renders the Web Component. To simplify things, the key data used for this -- the names `mfe2`, `./web-components` and `mfe2-root` -- are hard-coded in the example shown. In order to make the `WrapperComponent` universally applicable, it is advisable to make this information parameterizable, e. g. via an `@Input`:

```typescript
@Component([...])
export class WrapperComponent implements OnInit {
  elm = inject(ElementRef);

  @Input() config = initWrapperConfig;

  async ngOnInit() {
    const { exposedModule, remoteName, elementName } = this.config;
    
    await loadRemoteModule(remoteName, exposedModule);
    const root = document.createElement(elementName);
    this.elm.nativeElement.appendChild(root);
  }
}
```

The constant `initWrapperConfig` and its underlying type is defined as follows:

```typescript
export interface WrapperConfig {
    remoteName: string;
    exposedModule: string;
    elementName: string;
}

export const initWrapperConfig: WrapperConfig = {
    remoteName: '',
    exposedModule: '',
    elementName: '',
}
```

It's also noteworthy that since Angular 16, the router can directly bind routing parameters to an @Input. For this, activate the following feature during bootstrap:

```typescript
bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(
      APP_ROUTES, 
      withComponentInputBinding()
    )
  ],
});
```

This allows routes as the following:

```typescript
export const APP_ROUTES: Routes = [
  [...],
  {
    path: 'passengers',
    component: WrapperComponent,
    data: {
      config: {
        remoteName: 'mfe2',
        exposedModule: './web-components',
        elementName: 'mfe2-root',
      } as WrapperConfig,
    },
  },
  [...]´
];
```
## Sharing Zone.js

Angular currently still uses _Zone.js_ for change detection. The `NgZone` service represents -Zone.js_ within Angular. To avoid problems, you should ensure that all Micro Frontends and the shell use the same `NgZone` instance. To achieve this, the shell's `AppComponent` could share its `NgZone` across the global namespace:

```typescript
@Component([…])
export class AppComponent {
  constructor() {
      globalThis.ngZone = inject(NgZone);
  }
}
```

The individual Micro Frontends can then use this instance during bootstrapping:

```typescript
const app = await createApplication({
  providers: [
    globalThis.ngZone ? { provide: NgZone, useValue: globalThis.ngZone } : [],
    provideRouter(APP_ROUTES),
  ],
});
```

Fortunately, with Signals we are looking into a Zone-less future and once Angular works without _Zone.js_ we can git rid of this workaround.


## Web Components with own Routes

Things get a little more exciting when the micro frontend of the web component also uses routing. In this case, two routers duel over the URL - the shell's router and the Micro Frontend's router:

![Micro Frontend with its own routes](https://www.angulararchitects.io/wp-content/uploads/2023/10/router.png)

To ensure that the two routers do not interfere with each other, the following procedure has proven to be effective:

- Each route of the Micro Frontend is given a unique prefix.
- The shell tells its router to only pay attention to the first URL segment. Based on this segment, the shell loads a Micro Frontend, which makes its own routing decisions based on the remaining segments.

To specify which part of the Url is interesting for the respective router, you can use an `UrlMatcher`:


```typescript
[…]
import { loadRemoteModule } from '@angular-architects/native-federation';
import { WrapperComponent } from './wrapper/wrapper.component';
import { WrapperConfig } from './wrapper/wrapper-config';
import { startsWith } from './starts-with';

export const APP_ROUTES: Routes = [
  […]
  {
    matcher: startsWith('profile'),
    component: WrapperComponent,
    data: {
      config: {
        remoteName: 'mfe3',
        exposedModule: './web-components',
        elementName: 'mfe3-root',
      } as WrapperConfig,
    },
  },
  […]
];
```

The Angular router usually decides for or against a route based on the configured paths. `UrlMatchers` are an alternative to this. These are functions telling the router whether the configured route should be activated. The function [`startsWith`](https://github.com/manfredsteyer/module-federation-plugin-example/blob/nf-web-comp-mixed/projects/shell/src/app/starts-with.ts) for instance checks whether the current URL starts with the passed segment.

In our example, the shell's router uses this matcher to check whether the current URL starts with `profile`.

## Workaround for Routers in Web Component

In order for the router to react to route changes in the web component, it needs a special invitation. The examples discussed here include an helper method [`connectRouter`](https://github.com/manfredsteyer/module-federation-plugin-example/blob/nf-web-comp-mixed/projects/mfe3/src/app/connect-router.ts) that calls the Micro Frontend in its `AppComponent`:

```typescript
@Component({ … })
export class AppComponent implements OnInit {
  constructor() {
    connectRouter();
  }
}
```

## Conclusion

Combining several frameworks or framework versions is for sure not your first choice. However, if there is a good reason, you can achieve this goal by abstracting the different Micro Frontends. Using Web Components for this is a popular choice. 

However, we should be aware that no one is officially testing whether a given framework can peacefully coexist in the same browser window with another framework or another version of it self. Also, we need some workarounds, e.g. for the router or for sharing one Zone.js instance. The latter one is already counted, as Signals will eventually allow to go Zone-less.   

Another concern is increased bundle sizes. Lazy Loading different Micro Frontends with different frameworks or versions can help here. Also the next part of this series shows some approaches to improve performance in an Micro Frontend architecture.

# Micro Frontends with Modern Angular - Part 3: Island Architectures: A Chance for Highly Performant Micro Frontends?

Micro Frontends allow large frontends to be split into several smaller ones. As with Micro Services, the individual Micro Frontends can be implemented by different autonomous teams. A project can thus be scaled.

However, in the world of Single Page Applications (SPAs), the individual Micro Frontends, implemented and published separately, have to be loaded into the browser one by one. This is usually not a problem with business applications due to good bandwidth. Also, returning users benefit from the browser cache.

However, it is different with public web solutions such as web shops. Every millisecond and every kilobyte count here. After all, you want to keep the bounce rates of visitors low. For this reason, public web site often use Island Architectures. This part of our series describes how to implement Island Architectures for Angular-based Micro Frontends.

[Source Code](https://github.com/manfredsteyer/standalone-example-cli) 
(see branches `nf-web-comp-island`, `nf-web-comp-prerender`, and `nf-web-comp-ssr`)

## Islands

The basic idea behind Island Architectures is that a web page is composed of several interactive parts, the so-called islands: 

![Island Architectures](https://www.angulararchitects.io/wp-content/uploads/2023/10/island.png)

Different islands have different priorities. Initially, we just load the most important islands. Other islands are loaded on demand or at least at a later point in time. Interestingly, the priority of an island can change over time. For instance, the `Call to Action` island in the previous image might not be the browser windows's visible area (in the view port) after loading the page. Hence, initially, it's not important. However, this changes rapidly when the user scrolls it into view.

Before an island is loaded, the page might display a placeholder. In the easiest case, this is just an image indicating the structure of the island. Such images are called ghost images. It also could be a prerendered version of the island in the form of some static HTML. To prevent stale data in this HTML, we can render it on demand on the server-side. 

When loading the client-side version of the island, JavaScript code revives the static HTML. This process is called hydration. 

## Ghost Elements and Deferred Loading

The Wrapper Component used for loading Micro Frontends in the previous article has more potential than it seems at first glance. For example, we could instruct it to display a ghost element as a placeholder until the Micro Frontend loads:

```html
<div *ngIf="showPlaceholder">
    <img src="assets/ghost.gif" class="ghost">
</div>
```

In a further step, we could also delay loading the Micro Frontend until it is really needed. For example, to find out when the ghost element was scrolled into the view port, you can use an [Intersection Observer](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API). The following images demonstrate this idea using our example application.

![Ghost element as a placeholder for Micro Frontend](https://www.angulararchitects.io/wp-content/uploads/2023/10/ghost01.png)

![Micro Frontend replaces Ghost element](https://www.angulararchitects.io/wp-content/uploads/2023/10/ghost02.png)

First, the application displays the ghost element. Admittedly, I take the term "ghost element" very literally here. For demonstration purposes, the Intersection Observer has been configured to trigger an event when 75% of the ghost element is visible. In practice, this threshold is usually lower. When this event occurs, the `WrapperComponent` loads the Micro Frontends via Native Federation.

To make this happen, we can adjust the `WrapperComponent` as follows:

```typescript
@Component([...])
export class WrapperComponent implements OnInit {
  elm = inject(ElementRef);

  @Input() config = initWrapperConfig;

  showPlaceholder = true;

  async ngOnInit() {
    const options: IntersectionObserverInit = {
      root: null,
      threshold: 0.75,
    };

    const io = new IntersectionObserver((e) => {
      if (e[0].isIntersecting && this.showPlaceholder) {
        this.loadComponent();
        this.showPlaceholder = false;
      }
    }, options);

    io.observe(this.elm.nativeElement);
  }

  async loadComponent() {
    const { exposedModule, remoteName, elementName } = this.config;

    await loadRemoteModule(remoteName, exposedModule);
    const root = document.createElement(elementName);
    this.elm.nativeElement.appendChild(root);
  }

}
```

As an alternative to the generic ghost element, you could also load a pre-rendered version of the Micro Frontend. In order to better distinguish this from the actual Micro Frontend, I colored it gray for the example discussed here:

![Pre-rendered version as a ghost element](https://www.angulararchitects.io/wp-content/uploads/2023/10/ghost03.png)

The pre-rendered version of the Micro Frontend is static HTML that is loaded with the `HttpClient`:

```typescript
async loadFragment() {
  const {fragmentUrl, elementName } = this.config;

  const result = await lastValueFrom(this.http.get(fragmentUrl, { responseType: 'text' }));
  
  const html = extractFragment(result, elementName);
  const style = extractFragment(result, 'style');

  // Take care of XSS !!!
  this.elm.nativeElement.querySelector('#placeholder').innerHTML = `${style}\n${html}`;
}
```

With the [`extractFragment`](https://github.com/manfredsteyer/module-federation-plugin-example/blob/nf-web-comp-island-prerender/projects/shell/src/app/wrapper/wrapper-helper.ts) helper method I'm looking up the HTML element in question and its styles.

## Why not using @defer?

You might wonder why we don't use the new `@defer` feature introduced with Angular 17. The reason is, `@defer` works perfectly with Angular Components that are part of the same application. However, in our case, we want to load a Web Component from a different application. If we wanted to go with `@defer`, we could use it to load another `WrapperComponent` that loads the Web Component. However, this would require two HTTP calls in a row. To prevent this, our `WrapperComponent` directly takes care of deferred.

## Server-side Rendering of Micro Frontends

Before we extend our example for SSR, we need to find out how to use Federation on the server side. My current answer is: Don't. On the server-side it is far easier to directly embed the Markup from other Micro Frontends:

![Embedding HTML from Micro Frontends](https://www.angulararchitects.io/wp-content/uploads/2023/10/fragments.png)

This approach has several benefits:

- It's simple and huge companies rely on it (see below)
- No version conflicts, because each Micro Frontend is rendered in its very own process

The second benefit is especially interesting, because it brings back a benefit we have with Micro Services but not with SPA-based Micro Frontends: We don't care what happens when we call an URL as long as we get back the expected result. 

> There are also other takes on this that allow to run Federation on the server side and I want to express my greatest respect for all the people in this space working on this complicated topic and constantly pushing boundaries.

For loading the client-side JavaScript, we can still rely on Federation.

Interestingly, these idea is not new at all. It was already around before the term Micro Frontend was coined. For instance, [Ikea](https://www.infoq.com/news/2018/08/experiences-micro-frontends/), [Zalando](https://engineering.zalando.com/posts/2021/03/micro-frontends-part1.html), and [Otto](https://www.otto.de/jobs/en/technology/techblog/blogpost/frontends-with-microservices.php) have used this approach for their web shops for years. Being early adopters they have implemented their own frameworks for this. Also [DAZN](https://www.infoq.com/presentations/dazn-microfrontend/) is known to use this approach. Recently, this idea was also used by [Cloudflare](https://blog.cloudflare.com/better-micro-frontends) to demonstrate how high-performance Micro Frontends can be implemented with their stack.


## Native Federation and SSR with Hydration

For implementing our idea, we activate SSR and Hydration. For this, we just need to add the `@angular/ssr` package or use the `--ssr` switch when creating a new application (`ng new myProjecct --ssr`). Both possibilities are available since Angular 17. 

Next, the `WrapperComponent` gets a case distinction: On the server side, the shell loads an HTML fragment with the pre-rendered version of the Micro Frontend. It embeds this fragment into its template. On the client side, however, the `WrapperComponent` loads the Micro Frontend via Native Federation:

```typescript
if (isPlatformServer(this.platformId) && this.config.fragmentUrl) {
  this.loadFragment();
} else if (isPlatformBrowser(this.platformId)) {
  this.loadComponent();
}
```

Instead of directly loading the component, you can also defer loading as shown above.

Now that server-side rendered fragments from multiple applications end up on the same page, Angular needs to be able to map these fragments to individual applications. To do this, an `APP_ID` can be set for each application during bootstrapping, which Angular places in the pre-rendered HTML:

```typescript
export const appConfig: ApplicationConfig = {
  providers: [
    globalThis.ngZone ? { provide: NgZone, useValue: globalThis.ngZone } : [],
    { provide: APP_ID, useValue: 'mfe2' },
    provideClientHydration()
  ],
};
```

The updated `loadComponent` method also needs to extract the `script` tag with the JSON data islands containing the hydration data and transfer state. For this, `loadFragment` was extended:

```typescript
async loadFragment() {
  const { fragmentUrl, elementName, remoteName } = this.config;
  const result = await lastValueFrom(
    this.http.get(fragmentUrl, { responseType: 'text' })
  );

  const rawHtml = extractFragment(result, elementName);
  const style = extractFragment(result, 'style', 'ng-app-id', remoteName);
  const script = extractFragment(
    result,
    'script',
    'id',
    `${remoteName}-state`
  );

  const html = rawHtml;

  // Take care of XSS !!!
  this.elm.nativeElement.querySelector('#placeholder').innerHTML = `${style}\n${script}\n${html}`;
}
```

The now used version of [`extractFragment`](https://github.com/manfredsteyer/module-federation-plugin-example/blob/nf-web-comp-ssr/projects/shell/src/app/wrapper/wrapper-helper.ts) allows to extract a `script` tag with a given Id. The Id to look for is the defined `APP_ID` followed by `-state`. In our case the APP_ID is the `remoteName`, hence we look for `${remoteName}-state`.

## Native Federation and @angular/ssr

As of Angular 17, the new `@angular/ssr` package simplifies the use of SSR enormously. Native Federation also plays into this. In the simplest case, you install `@angular/ssr` first:

```bash
ng add @angular/ssr --project myProject

ng add @angular/native-federation --project myProject […]
```

However, if Native Federation has already been installed, it must first be temporarily removed:

```bash
ng g @angular/native- federation:remove --project myProject

ng add @angular/ssr --project myProject

ng add @angular/native-federation --project myProject […]
```

## Evaluation

Even if the approaches presented here are successfully used in practice by numerous companies, they are not without consequences. Both the transition from a monolith to a micro frontend and the use of SSR increase the complexity of the overall system:

![Evaluation of the approaches presented here](https://www.angulararchitects.io/wp-content/uploads/2023/10/bewertung.png)

That's why these two approaches and, above all, the combination of both will only be implemented if necessary. SSR is particularly essential for public solutions in the B2C sector. Every millisecond counts when loading the page to avoid bounces. Additionally, SSR helps with SEO.

Micro Frontends are often interesting when several autonomous teams are supposed to contribute to the overall system. As long as teams can work with a common monorepo, a well-structured monolith should be considered to reduce complexity.

## Summary

In order to improve the performance of public web solutions, the loading of Micro Frontends can be delayed. Instead of displaying the Micro Frontend, a ghost element can be displayed. As an alternative to the ghost element, we could go with a prerendered version of the Micro Frontend or a version that is rendered on the server side with current data. For the latter use case, Native Federation works together with `@angular/ssr`.
