# electron-window-redux

Use redux in electron, and control the action flow between main and browser process.

This project origins from [electron-redux](https://github.com/hardchor/electron-redux).
Before using this package, reading the original project is highly recommended.

## Features

- A `action` fired by main-process can be consumed by one or several specific renderer-processes.

- The `stores` of each process are individually created by redux. Main-process holds some universal state.

> One of the Three Redux Codes is `Single source of truth`, but javascript apps may have multi `process`, each process **DO NOT** share memory & resource, even with `ipc`, only serializable objects(eg json object) can be sent back and forth. Additionally, `main process` has no UI to render, so it is a better option use several stores, and inside each process, `Single Source of Truth` is still ought to be obeyed.   

- `windowManager` in main-process will use `name` and `windowID` property to identify windows with the same `name` or identify exact one window with `windowID`.


## Install

```
npm install --save electron-window-redux
```

`electron-window-redux` comes as redux middleware that is really easy to apply:

```javascript
// main store.js
import {
  forwardToRenderer
} from 'electron-window-redux';
import { createStore, applyMiddleware } from 'redux';
import rootReducer from './reducers';

// make sure store is singleton in the whole main-process.
const store = createStore(rootReducer, applyMiddleware(
  forwardToRenderer
));


export default store;

//==========

// main.js
import { app } from 'electron';
import { replayActionMain } from 'electron-window-redux';
import mainStore from './main/store';

app.on('ready', () => {
  // responce to actions from renderer-processes
  replayActionMain(mainStore);
}
```

```javascript
// in the renderer store
import {
  forwardToMain,
  replayActionRenderer,
} from 'electron-redux';

const rootReducer = combineReducers(reducers);

const store = createStore(
  rootReducer,
  initialState,
  applyMiddleware(
    forwardToMain, // IMPORTANT! This goes first
    ...otherMiddleware,
  )
);

replayActionRenderer(store);
```


## Actions

Actions fired **HAVE TO** be [FSA](https://github.com/acdlite/flux-standard-action#example)-compliant, i.e. have a `type` and `payload` property. Any actions not passing this test will be ignored and simply passed through to the next middleware.

> NB: `redux-thunk` is not FSA-compliant out of the box, but can still produce compatible actions once the async action fires.

### Local actions(renderer process)

By default, all actions are being broadcast from the renderer processes to the main processes. However, some state should only live in the renderer (e.g. `isPanelOpen`). `electron-redux` introduces the concept of action scopes.

To stop an action from propagating from renderer to main store, simply set the scope to `local`:

```javascript
function myLocalActionCreator() {
  return {
    type: 'MY_ACTION',
    payload: 123,
    meta: {
      scope: 'local',
    },
  };
}
```

Lifecycle of action with `local` scope:

`Renderer`: `store.dispatch`-> `Action` -> `store.reducers`

### Specified actions(renderer process & main process)

`electron-window-redux` use `name` and `windowID` to identify browserWindow.
(reminding that this windowID is generated by `Symbol` not the internal one created by `electron`).

eg: Set scope to `dialog` the action will be sent to browserWindows with name `dialog`, **no matter the action is created in
main process or renderer process**.

actions scope with `windowID` go the same way, except the only browserWindow with the same windowID will receive the action.


## API
***

### forwardToRenderer

Used as a redux middleware in **MAIN** process, filter actions dispatched by main store, send actions to renderer processes in accordance with attribute `action.meta.scope`.

```javascript
// main process
applyMiddleware(forwardToRenderer,...otherMiddlewares)
```

### replayActionMain

Dispatch actions received from renderers in **MAIN** process.

```javascript
// main process
app.on('ready', () => {
  // responce to actions from renderer-processes
  replayActionMain(mainStore);
}
```


### forwardToMain

Same function with `forwardToRenderer`, except using in **RENDERER** process and take actions from renderer to main.

```javascript
// renderer process
applyMiddleware(forwardToRenderer,...otherMiddlewares)
```

### replayActionRenderer

Same function with `replayActionMain`, except using in **RENDERER** process and dispatching actions to renderer process reducers.

Since middlewares or listeners above use Electron IPC module to communicate, so do not emit or listen any ipc events with `redux-action` tag.


### windowManager

* windowManager.add(window,name,onContentloaded)

  create a unique refrence to the input window, make the window under windowManager's control. 
  
  Add window before load url.
  ```javascript
  const stemWin = new BrowserWindow(WindowConfigs.stem);
  windowManager.add(stemWin, 'stem');
  stemWin.loadURL(STEM_INDEX_PATH);
  ```

  **Return**
  
  `Symbol(name)`, indicate the one and the only reference to `window`


  **Parameters**
  * `window`: BrowserWindow, a window instance
  * `name`: String, the name or category of the window, windows can have the same name.
  * `onContentloaded`： called when window content loaded *（optional）*

* windowManager.close(windowID)
  
  close corresponding window, delete reference


  **Parameters**
  * `windowID`: window id, it has to be the one returned by `windowManager.add`

* windowManager.get(windowID)
  
  get window instance by id
  
  **Parameters**
  * `windowID`: window id, it has to be the one returned by `windowManager.add`

* windowManager.getByInternalID(internalID)
  
  get window instance by internal id 
  
  **Parameters**
  * `internalID`: the origin property of BrowserWindow instace. `window.id`

* windowManager.getAll(name)

  get all window instances with the same name

  **Parameters**
  * `name`: the name passed in as parameter in `windowManager.add`.

* windowManager.subscribeWindowLoadedListener(fn)

  subscrible a listener to `'did-finish-load'` events of `BrowserWindow.webContents`.

  **Return**
  
  `function`, an unsubscrible function


  **Parameters**
  * `fn`: a listener will be called with `fn(windowId,name)`

* windowManager.subscribeWindowLoadedListener(fn)

  subscrible a listener to `'closed'` events of `BrowserWindow`.

  **Return**
  
  `function`, an unsubscrible function


  **Parameters**
  * `fn`: a listener will be called with `fn(windowId,name)`