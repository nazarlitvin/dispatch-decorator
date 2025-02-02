<p align="center">
  <img src="https://raw.githubusercontent.com/ngxs-labs/emitter/master/docs/assets/logo.png">
</p>

---

> The distribution for separation of concern between the state management and the view

[![NPM](https://badge.fury.io/js/%40ngxs-labs%2Fdispatch-decorator.svg)](https://www.npmjs.com/package/@ngxs-labs/dispatch-decorator)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](https://github.com/ngxs-labs/dispatch-decorator/blob/master/LICENSE)

This package simplifies the dispatching process. It would be best if you didn't care about `Store` service injection as we provide a more declarative way to dispatch events out of the box.

## 📦 Install

To install the `@ngxs-labs/dispatch-decorator`, run the following command:

```console
yarn add @ngxs-labs/dispatch-decorator
```

## 🔨 Usage

Import the module into your root application module:

```typescript
import { NgModule } from '@angular/core';
import { NgxsModule } from '@ngxs/store';
import { NgxsDispatchPluginModule } from '@ngxs-labs/dispatch-decorator';

@NgModule({
  imports: [NgxsModule.forRoot(states), NgxsDispatchPluginModule.forRoot()]
})
export class AppModule {}
```

### Dispatch Decorator

`@Dispatch()` is a function that allows you to decorate the methods and properties of your classes. Firstly let's create our state for demonstrating purposes:

```typescript
import { State, Action, StateContext } from '@ngxs/store';

export class Increment {
  static readonly type = '[Counter] Increment';
}

export class Decrement {
  static readonly type = '[Counter] Decrement';
}

@State<number>({
  name: 'counter',
  defaults: 0
})
export class CounterState {
  @Action(Increment)
  increment(ctx: StateContext<number>) {
    ctx.setState(ctx.getState() + 1);
  }

  @Action(Decrement)
  decrement(ctx: StateContext<number>) {
    ctx.setState(ctx.getState() - 1);
  }
}
```

We are ready to try the plugin after registering our state in the `NgxsModule`, given the following component:

```typescript
import { Component } from '@angular/core';
import { Select } from '@ngxs/store';
import { Dispatch } from '@ngxs-labs/dispatch-decorator';

import { Observable } from 'rxjs';

import { CounterState, Increment, Decrement } from './counter.state';

@Component({
  selector: 'app-root',
  template: `
    <ng-container *ngIf="counter$ | async as counter">
      <h1>{{ counter }}</h1>
    </ng-container>

    <button (click)="increment()">Increment</button>
    <button (click)="decrement()">Decrement</button>
  `
})
export class AppComponent {
  @Select(CounterState) counter$: Observable<number>;

  @Dispatch() increment = () => new Increment();

  @Dispatch() decrement = () => new Decrement();
}
```

As you may mention, we don't have to inject the `Store` class to dispatch those actions. The `@Dispatch` decorator does it for you underneath. It gets the result of the function call and invokes `store.dispatch(...)` under the hood.

Dispatchers can also be asynchronous. They can return either `Promise` or `Observable`. Asynchronous operations are handled outside of Angular's zone; thus it doesn't affect performance:

```typescript
export class AppComponent {
  // `ApiService` is defined somewhere
  constructor(private api: ApiService) {}

  @Dispatch()
  async setAppSchema() {
    const version = await this.api.getApiVersion();
    const schema = await this.api.getSchemaForVersion(version);
    return new SetAppSchema(schema);
  }

  // OR using lambda

  @Dispatch() setAppSchema = () =>
    this.api.getApiVersion().pipe(
      mergeMap(version => this.api.getSchemaForVersion(version)),
      map(schema => new SetAppSchema(schema))
    );
}
```

Notice that it doesn't matter if you use an arrow function or a regular class method.

### Dispatching Multiple Actions

Dispatchers can return arrays. NGXS will handle actions synchronously if their action handlers do a synchronous job and vice versa if their handlers are asynchronous.

```typescript
export class AppComponent {
  @Dispatch() setLanguageAndNavigateHome = (language: string) => [
    new SetLanguage(language),
    new Navigate('/')
  ];
}
```

### Canceling

If you have an async dispatcher, you may want to cancel a previous `Observable` if the dispatcher has been invoked again. This is useful for cancelling previous requests like in a typeahead. Given the following example:

```ts
@Component({ ... })
export class NovelsComponent {
  @Dispatch() searchNovels = (query: string) =>
    this.novelsService.getNovels(query).pipe(map(novels => new SetNovels(novels)));

  constructor(private novelsService: NovelsService) {}
}
```

If we want to cancel previously uncompleted `getNovels` request, then we need to provide the `cancelUncompleted` option:

```ts
@Component({ ... })
export class NovelsComponent {
  @Dispatch({ cancelUncompleted: true }) searchNovels = (query: string) =>
    this.novelsService.getNovels(query).pipe(map(novels => new SetNovels(novels)));

  constructor(private novelsService: NovelsService) {}
}
```
