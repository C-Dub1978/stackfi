# web/ — Stackfi frontend (Angular 22 + HMHA)

These rules apply to everything under `web/`. They add to the root `CLAUDE.md`; they don't repeat it. In `web/` you work in normal pair-programming mode: write and edit code freely.

You are an expert in TypeScript, Angular 22 and accessible, performant web apps. Write small, focused, standalone, signal-based code.

## Tech stack

- **Framework:** Angular 22: standalone, zoneless, signals. NEVER downgrade `@angular/core` or `@angular/cli`.
- **UI:** the HMHA design system: `@halfmanhalfape/hmha-ui` (components) and `@halfmanhalfape/hmha-tokens` (design tokens), version 1.2.0 or later.
  - Do not add PrimeNG, Angular Material or any other component library without asking.
- **Playback:** hls.js. **Uploads:** Uppy 6, headless (`@uppy/core` + `@uppy/aws-s3`).
- **API:** the TypeScript client generated from the API's OpenAPI spec. Don't hand-write request or response types.

## Folder structure

```
src/app/
├── core/        app-wide singletons: auth, API client config, interceptors, guards, theme
├── shell/       app shell: top bar, side nav, router outlet layout
├── features/    one folder per feature: catalog, player, library, upload, search, playlists, account
└── shared/      reusable Stackfi UI built from HMHA: media card, empty/error states, formatters
```

Keep each component in its own folder inside its feature.

## Files and naming

- For a component named `ProjectDashboard`, create exactly three files, in **kebab-case** with no `.component` suffix:
  - `project-dashboard.ts`
  - `project-dashboard.html`
  - `project-dashboard.css`
- **The class name drops the suffix too:** `export class ProjectDashboard`, not `ProjectDashboardComponent`. The same goes for directives and pipes.
- **Use plain CSS, not SCSS.** HMHA is token-driven CSS, and native CSS nesting covers what SCSS was used for.
- **Selector prefix is `sf-`** (for example `selector: 'sf-project-dashboard'`). Never use `app-`.

## HMHA design system

- `angular.json` styles must load `node_modules/@halfmanhalfape/hmha-tokens/src/lib/_all.css` **first**, before `src/styles.css`.
- Most HMHA components are **attribute directives on native elements**: `<button hmhaButton>`, `<input hmhaInput>`, `<progress hmhaProgressBar>`, `<a hmhaNavItem>`. Import the class (`HmhaButton`) into the component's `imports` array. There are no NgModules and no providers.
  - Only `hmha-card`, `hmha-field`, `hmha-icon` and `hmha-spinner` are custom tags. Toast is a service.
- Before using a component, check its README in the library (`libs/ui/src/lib/<component>/README.md`) for its inputs and component tokens. Don't guess input names.
- **Styling uses tokens only.**
  - Use `var(--hmha-*)` semantic tokens for colour, spacing, radius, type and motion. No hex values, raw colour functions or stray px values.
  - Customise an HMHA component by setting its **component tokens** (for example `--hmha-button-bg`). Never use `::ng-deep` and never override its internal classes.
  - If a value you need has no token, ask. Don't hard-code it.
- **Theming** uses attributes on `<html>`: `data-hmha-mode="light|dark"` and `data-hmha-density="comfortable|compact"`.
  - When `data-hmha-mode` is unset, the OS preference applies.
  - A `ThemeService` in `core/` owns these attributes.
- **Keep Stackfi-specific UI in Stackfi; compose it from HMHA primitives.** This covers the app shell, media cards, player controls, the mini audio player, the upload queue and (later) the book reader. If something seems generic enough to belong in HMHA, suggest it; don't add it to the library from here.
- **Navigation:** the HMHA nav list is router-agnostic. Add the routing yourself:

  ```html
  <a hmhaNavItem routerLink="/library" routerLinkActive ariaCurrentWhenActive="page">My Library</a>
  ```

## Component rules

- **Don't write defaults:**
  - No `standalone: true`; standalone is the default.
  - No `changeDetection: ChangeDetectionStrategy.OnPush`; it's the default for components in Angular 22.
  - No `provideZonelessChangeDetection()` beyond what the CLI generated; zoneless is the default for new apps.
- Use **`inject()`** only. No constructor injection.
- Use **signal APIs** only: `input()`, `input.required()`, `output()`, `model()`, `viewChild()`, `contentChildren()`.
  - Never use `@Input`, `@Output`, `@ViewChild` or `EventEmitter`.
- Put **host bindings** in the `host: {}` object. Never use `@HostBinding` or `@HostListener`.
- Use **native bindings** (`[class.x]`, `[style.x]`, `[attr.x]`). Never use `ngClass` or `ngStyle`.
- Use **built-in control flow** (`@if`, `@for` with `track`, `@switch`, `@defer`). Never use `*ngIf` or `*ngFor`.
- **Images:** use `NgOptimizedImage` (`ngSrc`) for posters, album art and avatars, including signed CDN URLs. Always give a `width` and `height`, or use `fill`. It doesn't support inline base64 images.
- **Zoneless safety:** never rely on `setTimeout`, `NgZone` or zone patching to update the view. Change state through signals.

## Smart and presentation components

- **Smart (container) components** inject services, hold state, run resources and handle events. Keep their templates and styling minimal.
- **Presentation components** only display things. They take data through `input()`, send actions up through `output()`, and MUST NOT inject services, stores or API clients. Most of `shared/` and the player controls are presentation components.

## State and async

- **Local state:**
  - Use `signal()` for state, and `computed()` or `linkedSignal()` for derived values.
  - Update with `set()` or `update()`; `mutate()` doesn't exist.
- **Reading data** (GET requests, anything a view displays) uses Angular's resource APIs:
  - `rxResource()` wraps calls from the generated client, which return Observables.
  - `httpResource()` is for simple one-off GET requests.
  - `resource()` is for other async reads.
  - Pass reactive inputs through the **`params`** option. The old `request` key is deprecated.
  - Use a resource's own status signals (`isLoading()`, `error()`, `value()`) to drive HMHA skeletons, empty states and alerts.
- **Writing data** (POST, PUT, PATCH, DELETE: login, start upload, add to playlist) does **not** use a resource. Resources are for reads.
  - Call the client from a method, convert the result with `firstValueFrom()`, and update local state when it returns.
  - Disable the triggering button while the call is in flight.
- **Prefer `computed()` to `effect()`.** Use `effect()` only for side effects that sync state outward: DOM attributes, `localStorage`, imperative libraries such as hls.js or Uppy. Use `afterRenderEffect()` when the effect needs the rendered DOM.
- **Effects must read a signal.** An `effect()` that reads no signal runs once and never again.

  ```ts
  // BAD: reads no signal, so it runs once and stops tracking
  effect(() => {
    document.documentElement.setAttribute('data-hmha-mode', 'dark');
  });

  // GOOD: reading mode() makes the effect re-run whenever the mode changes
  effect(() => {
    document.documentElement.setAttribute('data-hmha-mode', this.mode());
  });
  ```

## Routing: no RxJS subscriptions

- Never use `.subscribe()` for route parameters, query parameters or form values.
- **Preferred:** enable `withComponentInputBinding()` in `provideRouter()`, so route and query parameters arrive as `input()` signals:

  ```ts
  // route: { path: 'title/:itemId', component: TitleDetail }
  export class TitleDetail {
    readonly itemId = input.required<string>();
  }
  ```

- **Otherwise,** wrap the observable with `toSignal()`: `toSignal(inject(ActivatedRoute).paramMap)`.
- Lazy-load every feature: `loadComponent` for single components, and `loadChildren` pointing to a routes file for feature areas.

## Forms

- **All forms use Signal Forms** (`@angular/forms/signals`) and bind controls with `[formField]`. Never use template-driven or reactive forms in new code.
- **HMHA form controls work with Signal Forms.** `HmhaInput`, `HmhaCheckbox`, `HmhaSwitch`, `HmhaSelect` and the other controls implement `ControlValueAccessor`, which `[formField]` supports for compatibility:

  ```html
  <input hmhaInput [formField]="form.email" />
  ```

  If one doesn't bind correctly, report it; don't fall back to reactive forms.
- Wrap each control in `<hmha-field>` for its label, hint and error text. Show validation errors from the Signal Forms field state.

## Auth

The full design is in the root `CLAUDE.md`, decision 6.

- **Access token:** kept in memory only, in a signal in `AuthService`. Never in `localStorage`.
- **Refresh token:** an HttpOnly cookie on the web, which JavaScript never sees. In Phase 10 it moves to the device keychain under Capacitor.
- **Silent refresh at startup:** use `provideAppInitializer()` to try a refresh before the first navigation. Without it, a page reload looks logged-out until the refresh call returns.
- **An HTTP interceptor (`HttpInterceptorFn`)** adds the bearer token. On a 401 it refreshes once and retries, sharing a single in-flight refresh between concurrent requests.
- **Guards are functional `CanActivateFn`s** that read auth state from signals. Return a `UrlTree`; don't call `router.navigate()` inside the guard:

  ```ts
  export const authGuard: CanActivateFn = () => {
    const auth = inject(AuthService);
    return auth.isAuthenticated() || inject(Router).createUrlTree(['/sign-in']);
  };
  ```

## Media

- **hls.js** lives in a player service or directive, not in components.
  - If `Hls.isSupported()`, attach hls.js. Otherwise, if `video.canPlayType('application/vnd.apple.mpegurl')` succeeds (Safari, iOS), set the source directly.
  - Always call `hls.destroy()` on teardown, through `DestroyRef`.
- **Playback progress:** send it to `PUT /api/progress/{itemId}` about every 10 seconds, on pause, and on page hide (`visibilitychange`). The body is a `locator` object, not a raw number.
- **Uploads:**
  - `HmhaFileDropzone` emits `File[]` and makes no network requests.
  - `core/upload/UploadService` adds the files to Uppy (`uppy.addFile`) and turns Uppy's state into signals.
  - `HmhaUploadList` and `HmhaUploadItem` with `HmhaProgressBar` render that state.
  - Leave the dropzone's own `accept`, `maxSize` and `maxFiles` unset. Uppy's restrictions are the single source of rules.
  - Uppy uses the API to sign every request. Never put AWS credentials in the browser.
  - An upload row shows the whole journey: uploading, then processing (indeterminate), then ready or failed.

## Accessibility

- Every control has an accessible name. Icon-only buttons have a `label`, and action buttons in lists include the item ("Remove movie.mkv").
- Use HMHA's native-element components rather than clickable `div`s.
- Announce async results (upload finished, item added to playlist) through HMHA Toast or a live region.
- Respect `prefers-reduced-motion` in any custom animation.

## Testing

- Use the test runner the Angular CLI configured for this workspace. Don't switch runners without asking.
- Test presentation components through their inputs and outputs, and smart components with a mocked API client.
- For HMHA attribute-selector components, test them in a host component template that uses the real native element (`<button hmhaButton>`). `TestBed.createComponent()` on the directive alone renders a `<div>`.
