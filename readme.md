# Tutorial: Getting Started with Webpack Module Federation and Angular

This tutorial shows how to use Webpack Module Federation together with the Angular CLI and the `@angular-architects/module-federation` plugin. The goal is to make a shell capable of **loading a separately compiled and deployed microfrontend**:

![Microfrontend Loaded into Shell](https://github.com/angular-architects/module-federation-plugin/raw/main/libs/mf/tutorial/result.png)

## Part 1: Clone and Inspect the Starter Kit

In this part you will clone the starter kit and inspect its projects.

1. **If not already done during the preparation,** Clone the starter kit for this tutorial:

   ```
   git clone https://github.com/manfredsteyer/module-federation-plugin-example.git --branch starter
   ```

2. **If not already done during the preparation,** switch into the project directory and install the dependencies **with npm**:

   ```
   cd module-federation-plugin-example
   npm i
   ```

3. Start the shell (`ng serve shell -o`) and inspect it a bit:

   1. Click on the `flights` link. It leads to a dummy route. This route will later be used for loading the separately compiled Micro Frontend.

   2. Have a look to the shell's source code.

   3. Stop the CLI (`CTRL+C`).

4. Do the same for the Micro Frontend. In this project, it's called `mfe1` (Micro Frontend 1) You can start it with `ng serve mfe1 -o`.

## Part 2: Activate and Configure Module Federation

Now, let's activate and configure module federation:

1. Install `@angular-architects/module-federation` into the shell:

   ```
   ng add @angular-architects/module-federation --project shell --type host --port 4200
   ```

   Also, install it into the micro frontend:

   ```
   ng add @angular-architects/module-federation --project mfe1 --type remote --port 4201
   ```

   This activates module federation, assigns a port for ng serve, and generates the skeleton of a module federation configuration.

2. Switch into the project `mfe1` and open the generated configuration file `projects\mfe1\webpack.config.js`. It contains the module federation configuration for `mfe1`. Adjust it as follows:

   ```javascript
   const {
     shareAll,
     withModuleFederationPlugin,
   } = require('@angular-architects/module-federation/webpack');

   module.exports = withModuleFederationPlugin({
     name: 'mfe1',

     exposes: {
       // Update this whole line (both, left and right part):
       './Module': './projects/mfe1/src/app/flights/flights.module.ts',
     },

     shared: {
       ...shareAll({
         singleton: true,
         strictVersion: true,
         requiredVersion: 'auto',
       }),
     },
   });
   ```

   This exposes the `FlightsModule` under the Name `./Module`. Hence, the shell can use this path to load it.

3. Switch into the `shell` project and open the file `projects\shell\webpack.config.js`. Make sure, the mapping in the remotes section uses port `4201` (and hence, points to the Micro Frontend):

   ```javascript
   const {
     shareAll,
     withModuleFederationPlugin,
   } = require('@angular-architects/module-federation/webpack');

   module.exports = withModuleFederationPlugin({
     remotes: {
       // Check this line. Is port 4201 configured?
       'mfe1': 'http://localhost:4201/remoteEntry.js',
     },

     shared: {
       ...shareAll({
         singleton: true,
         strictVersion: true,
         requiredVersion: 'auto',
       }),
     },
   });
   ```

   This references the separately compiled and deployed `mfe1` project.

4. Open the `shell`'s router config (`projects\shell\src\app\app.routes.ts`) and add a route loading the Micro Frontend:

   ```javascript
   {
      path: 'flights',
      loadChildren: () => import('mfe1/Module')
        .then(m => m.FlightsModule)
   },

   {
      path: '**',
      component: NotFoundComponent
   }

   // DO NOT insert routes after this one.
   // { path:'**', ...} needs to be the LAST one.
   ```

   Please note that the imported URL consists of the names defined in the configuration files above.

5. As the URL `mfe1/Module` does not exist at compile time, ease the TypeScript compiler by adding the following line to the file `projects\shell\src\decl.d.ts`:

   ```javascript
   declare module 'mfe1/Module';
   ```

> **Solution for this lab:** Please find the solution for this lab in the branch ``ngconf-2022-safepoint-step2``.

## Part 3: Try it out

Now, let's try it out!

1. Start the `shell` and `mfe1` side by side in two different terminals:

   ```
   ng serve shell -o
   ng serve mfe1 -o
   ```

   **Hint:** You might use two terminals for this.

2. After a browser window with the shell opened (`http://localhost:4200`), click on `Flights`. This should load the Micro Frontend into the shell:

   ![Shell](https://github.com/angular-architects/module-federation-plugin/raw/main/libs/mf/tutorial/shell.png)

3. Also, ensure yourself that the Micro Frontend also runs in standalone mode at http://localhost:4201:

   ![Microfrontend](https://github.com/angular-architects/module-federation-plugin/raw/main/libs/mf/tutorial/mfe1.png)

**Hint:** You can also call the following script to start all projects at once: `npm run run:all`. This script is added by the Module Federation plugin.

**Congratulations!** You've implemented your first Module Federation project with Angular!

## Part 4: Switch to Dynamic Federation

Now, let's remove the need for registering the Micro Frontends upfront with with shell.

### Part 4a: Basic Usage of Dynamic Federation

1. Switch to your `shell` application and open the file `projects\shell\webpack.config.js`. Here, remove the registered remotes:

   ```javascript
   remotes: {
      // Remove this line:
      // "mfe1": "http://localhost:4201/remoteEntry.js",
   },
   ```

2. Open the file `app.routes.ts` and use the function `loadRemoteModule` instead of the dynamic `import` statement:

   ```typescript
   import { loadRemoteModule } from '@angular-architects/module-federation';

   [...]
   const routes: Routes = [
      [...]
      {
          path: 'flights',
          loadChildren: () =>
               loadRemoteModule({
                  type: 'module',
                  remoteEntry: 'http://localhost:4201/remoteEntry.js',
                  exposedModule: './Module'
              })
              .then(m => m.FlightsModule)
      },
      [...]
   ]
   ```

   _Remarks:_ `type: 'module'` is needed for Angular 13 or higher as beginning with version 13, the CLI emits EcmaScript modules instead of "plain old" JavaScript files.

3. **Restart both**, the `shell` and the micro frontend (`mfe1`).

4. The shell should still be able to load the micro frontend. However, now it's loaded dynamically.

> **Solution for this lab:** Please find the solution for this lab in the branch ``ngconf-2022-safepoint-step4a``.


### Part 4b: Loading Meta Data Upfront

This was quite easy, wasn't it? However, we can improve this solution a bit. Ideally, we load the Micro Frontend's **remoteEntry.js** before Angular bootstraps. This file contains meta data about the Micro Frontend, esp. about its shared dependencies. Knowing about them upfront helps Module Federation to avoid version conflicts.

1. Switch to the `shell` project and open the file `main.ts`. Adjust it as follows:

    ```typescript
    import { loadRemoteEntry } from '@angular-architects/module-federation';

    Promise.all([
      loadRemoteEntry({
        type: 'module',
        remoteEntry: 'http://localhost:4201/remoteEntry.js',
      }),
    ])
    .catch((err) => console.error('Error loading remote entries', err))
    .then(() => import('./bootstrap'))
    .catch((err) => console.error(err));
    ```

2. Restart both, the `shell` and the micro frontend (`mfe1`).

3. The shell should still be able to load the micro frontend.

> **Solution for this lab:** Please find the solution for this lab in the branch ``ngconf-2022-safepoint-step4b``.


## Step 5: Communication Between Micro Frontends and Sharing Monorepo Libraries

1. Add a library to your monorepo:

    ```
    ng g lib auth-lib
    ```

2. In your `tsconfig.json` in the workspace's root, adjust the path mapping for `auth-lib` so that it points to the libs entry point:

    ```json
    "auth-lib": [
        "projects/auth-lib/src/public-api.ts"
    ]
    ```

3. As most IDEs only read global configuration files like the `tsconfig.json` once, restart your IDE (Alternatively, your IDE might also provide an option for reloading these settings).

4. Switch to your `auth-lib` project and open the file `auth-lib.service.ts` (`projects\auth-lib\src\lib\auth-lib.service.ts`). Adjust it as follows:

    ```typescript
    import { Injectable } from '@angular/core';

    @Injectable({
      providedIn: 'root',
    })
    export class AuthLibService {
      private userName: string;

      public get user(): string {
        return this.userName;
      }

      constructor() {}

      public login(userName: string, password: string): void {
        // Authentication for **honest** users TM. (c) Manfred Steyer
        this.userName = userName;
      }
    }
    ```

5. Switch to your `shell` project and open its `app.component.ts` (`projects\shell\src\app\app.component.ts`). Use the `AuthLibService` to login a user:

    ```typescript
    [...]

    // IMPORTANT: Make sure you import the service
    //  from 'auth-lib'!
    import { AuthLibService } from 'auth-lib';

    [...]

    @Component({
      selector: 'app-root',
      templateUrl: './app.component.html',
    })
    export class AppComponent {
      title = 'shell';

      constructor(private service: AuthLibService) {
        this.service.login('Max', null);
      }
    }
    ```

6. Switch to your `mfe1` project and open its `flights-search.component.ts` (``projects\mfe1\src\app\flights\flights-search\flights-search.component.ts``). Use the shared service to retrieve the current user's name:

    ```typescript
    [...]

    // IMPORTANT: Make sure you import the service
    //  from 'auth-lib'!
    import { AuthLibService } from 'auth-lib';

    [...]

    export class FlightsSearchComponent {

        // Add this:
        user = this.service.user;

        // And add that:
        constructor(private service: AuthLibService) { }

        [...]
    }
    ```

7. Open this component's template(`flights-search.component.html`) and data bind the property `user`:

    ```html
    <div id="container">

      <!-- Add this line: -->
      <div>User: {{user}}</div>
      [...]
    </div>
    ```

8. Restart both, the `shell` and the micro frontend (`mfe1`).

9. In the shell, navigate to the micro frontend. If it shows the used user name `Max`, the library is shared.

**Remarks:** All the libraries of your Monorepo are shared by default. The next section shows how to select libraries to share.

> **Solution for this lab:** Please find the solution for this lab in the branch ``ngconf-2022-safepoint-step5``.


## Bonus: Loading Web Components for Multi-Framework-Scenarios

1. Install `@angular-architects/module-federation-tools`:

   ```
   npm i @angular-architects/module-federation-tools
   ```

2. Restart your VS Code (or your TS Server within VS Code at least)

3. Open your shell's `app.routes.ts` and add the following routes:

    ```typescript
    [...]

    // Add this import:
    import { WebComponentWrapper, WebComponentWrapperOptions, startsWith } from '@angular-architects/module-federation-tools';
    
    [...]


    export const APP_ROUTES: Routes = [

        [...]
        // Add these routes:
        {
                path: 'react',
                component: WebComponentWrapper,
                data: {
                    type: 'script',
                    remoteEntry: 'https://witty-wave-0a695f710.azurestaticapps.net/remoteEntry.js',
                    remoteName: 'react',
                    exposedModule: './web-components',
                    elementName: 'react-element'
                } as WebComponentWrapperOptions
        },

        {
                path: 'angular1',
                component: WebComponentWrapper,
                data: {
                    type: 'script',
                    remoteEntry: 'https://nice-grass-018f7d910.azurestaticapps.net/remoteEntry.js',
                    remoteName: 'angular1',
                    exposedModule: './web-components',
                    elementName: 'angular1-element'
                } as WebComponentWrapperOptions
        },

        {
                path: 'angular2',
                component: WebComponentWrapper,
                data: {
                    type: 'script',
                    remoteEntry: 'https://gray-pond-030798810.azurestaticapps.net//remoteEntry.js',
                    remoteName: 'angular2',
                    exposedModule: './web-components',
                    elementName: 'angular2-element'
                } as WebComponentWrapperOptions
        },

        {
            matcher: startsWith('angular3'),
            component: WebComponentWrapper,
            data: {
                type: 'script',
                remoteEntry: 'https://gray-river-0b8c23a10.azurestaticapps.net/remoteEntry.js',
                remoteName: 'angular3',
                exposedModule: './web-components',
                elementName: 'angular3-element'
            } as WebComponentWrapperOptions
        },

        {
            path: 'vue',
            component: WebComponentWrapper,
            data: {
                type: 'script',
                remoteEntry: 'https://mango-field-0d0778c10.azurestaticapps.net/remoteEntry.js',
                remoteName: 'vue',
                exposedModule: './web-components',
                elementName: 'vue-element'
            } as WebComponentWrapperOptions
        },

        {
            path: 'angularjs',
            component: WebComponentWrapper,
            data: {
                type: 'script',
                remoteEntry: 'https://calm-mud-0a3ee4a10.azurestaticapps.net/remoteEntry.js',
                remoteName: 'angularjs',
                exposedModule: './web-components',
                elementName: 'angularjs-element'
            } as WebComponentWrapperOptions
        },

        {
            matcher: startsWith('angular3'),
            component: WebComponentWrapper,
            data: {
                type: 'script',
                remoteEntry: 'https://gray-river-0b8c23a10.azurestaticapps.net/remoteEntry.js',
                remoteName: 'angular3',
                exposedModule: './web-components',
                elementName: 'angular3-element'
            } as WebComponentWrapperOptions
        },


        // THIS needs to be the last route!!!
        {
            path: '**',
            component: NotFoundComponent
        }

    ];
    ```

   **Remarks:** The URL matcher ``startsWith`` makes the shell to ignore the remaining part of the URL. This is necessary when the loaded micro frontend uses a router too.

   **Remarks:** Please note that we are using `type: 'script'` here. This is needed for classic webpack setups as normally used in the Vue and React world as well as for Angular before version 13. Beginning with version 13, the CLI emits EcmaScript module instead of "plain old" JavaScript files. Hence, when loading a remote compiled with Angular 13 or higher, you need to set `type` to `module`. In our case, however, the remotes we find at the shown URLs in the cloud are Angular 12-based, hence we need `type: 'script'`.

4. Open your shell's `app.component.html` and add the following links:

   ```html
   <!-- Add these links -->
   <li><a routerLink="/react">React</a></li>
   <li><a routerLink="/angular1">Angular 1</a></li>
   <li><a routerLink="/angular2">Angular 2</a></li>
   <li><a routerLink="/angular3/a">Angular 3</a></li>
   <li><a routerLink="/vue">Vue</a></li>
   <li><a routerLink="/angularjs">AngularJS</a></li>
   ```

5. Open your shell's `bootstrap.ts` and use the `bootstrap` helper function found in `@angular-architects/module-federation-tools` for bootstrapping:

    ```typescript
    import { AppModule } from './app/app.module';
    import { environment } from './environments/environment';
    import { bootstrap } from '@angular-architects/module-federation-tools';

    bootstrap(AppModule, {
        production: environment.production,
        appType: 'shell',
    });
    ```

    Remarks: This special bootstrap function takes care of some **workarounds** necessary to run several versions of Angular side by side.

6. Start your ``shell`` and the ``mfe1`` project (e. g. by calling `npm run run:all`) and try it out.

## Bonus: Inspect the Web-Component-based Micro Frontends

In this part of the lab, we will investigate the loaded micro frontend that has been called ["MF Angular #3"](https://github.com/manfredsteyer/angular3-app) before. We want to draw your attention to the following details:

1. The application is bootstrapped with the [bootstrap function](https://github.com/manfredsteyer/angular3-app/blob/main/src/bootstrap.ts) already used above. Please note that here, `appType` is set to `microfrontend`.

2. The `AppModule` is wrapping some components as web components using Angular Elements in it's [ngDoBootstrap](https://github.com/manfredsteyer/angular3-app/blob/main/src/app/app.module.ts) method.

3. The [webpack config](https://github.com/manfredsteyer/angular3-app/blob/main/webpack.config.js) exposes the whole `bootstrap.ts` file. Hence, everyone importing it can use the provided web components.

4. The [webpack config](https://github.com/manfredsteyer/angular3-app/blob/main/webpack.config.js) shares libraries like `@angular/core`.

### Bonus: Using a Registry

So far, we just hardcoded the URLs pointing to our Micro Frontends. However, in a real world scenario, we would rather get this information at runtime from a config file or a registry service. This is what this part of the lab is about.

1. Switch to the shell and create a file `mf.manifest.json` in its `assets` folder (`projects\shell\src\assets\mf.manifest.json`):

  ```json
  {
    "mfe1": "http://localhost:4201/remoteEntry.js"
  }
  ```

2. Adjust the shell's `main.ts` (`projects/shell/src/main.ts`) as follows:

    ```typescript
    import { loadManifest } from '@angular-architects/module-federation';

    loadManifest('assets/mf.manifest.json')
        .catch((err) => console.error('Error loading remote entries', err))
        .then(() => import('./bootstrap'))
        .catch((err) => console.error(err));
    ```

  The imported `loadManifest` function also loads the remote entry points.

3. Adjust the shell's lazy route pointing to the Micro Frontend as follows (`projects/shell/src/app/app.routes.ts`):

  ```typescript
  {
      path: 'flights',
      loadChildren: () =>
          loadRemoteModule({
              type: 'manifest',
              remoteName: 'mfe1',
              exposedModule: './Module'
          })
          .then(m => m.FlightsModule)
  },
  ```

4. Restart both, the `shell` and the micro frontend (`mfe1`).

5. The shell should still be able to load the micro frontend.

**Hint:** The `ng add` command used initially also provides an option `--type dynamic-host`. This makes ``ng add`` to generate the `mf.manifest.json` as well as the call to `loadManifest` in the `main.ts`.

> **Solution for this lab:** Please find the solution for this lab in the branch ``ngconf-2022-safepoint-step4c``.


## More Details on Module Federation

Have a look at this [article series about Module Federation](https://www.angulararchitects.io/aktuelles/the-microfrontend-revolution-part-2-module-federation-with-angular/)

## Angular Trainings, Workshops, and Consulting

- [Angular Trainings and Workshops](https://www.angulararchitects.io/en/angular-workshops/)
- [Angular Consulting](https://www.angulararchitects.io/en/consulting/)
