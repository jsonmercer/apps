Apps programming model


![diagram](https://p-qkfgo2.t2.n0.cdn.getcloudapp.com/items/E0u4vmOj/f4dbe333-9203-4628-b3d0-0092af5f357e.png)
[Link to image](https://p-qkfgo2.t2.n0.cdn.getcloudapp.com/items/E0u4vmOj/f4dbe333-9203-4628-b3d0-0092af5f357e.png)

An App Platform App is defined by an **App Manifest** created and maintained on the Developer Platform. The App Manifest defines top-level metadata about an app, including its name, logo, the url of the **Main IFrame** controller. In addition, it defines the set of **Features** enabled for the app, along with each feature’s relevant configuration, and the set of **Scopes** allowed when accessing the [Datadog public API](https://docs.datadoghq.com/api/).

**Features** define what the app can do in the Datadog UI (for example, provide a Custom Widget or render custom Cog Menu Items). Each feature will require a different set of metadata. In some cases this metadata may be sufficient for Datadog to implement the customization on behalf of the app: for example, we can add a cog menu item based on a list of items provided in the **App Manifest**. In other cases the feature may require additional implementation by the developer in an **IFrame**. For example: custom widgets are rendered by an IFrame implemented and hosted by the developer. Only some light top-level config such as the url of the custom widget IFrame are configured in the App Manifest.

Most nontrivial apps will involve at least one **IFrame** implementing custom logic and UI elements. Each Iframe must include a copy of the Official [JavaScript SDK](https://www.npmjs.com/package/@datadog/ui-apps-sdk). The **SDK** is responsible for all communication with the Datadog UI, including retrieving context data, listening for and dispatching events, and accessing the Datadog public API within the app’s granted **Scopes**.

Many Apps will involve multiple **IFrames** mounted within the UI in different places. For example: Consider an app that implements a custom widget, with a custom context menu item that triggers a modal to open with arbitrary custom content. This app would involve three IFrames: A single Frame would render the custom widget content based on dashboard context provided through the **SDK**. A **Main IFrame** would be required to listen in the background for user events from the **Custom Context Menu** (for example, a click event with information about the clicked item). A third frame would render whatever content the developer chooses in the modal.

This unusual programming model requires inter-iframe communication as a first-class feature. This is handled entirely with through the **SDK**.

## App Manifest

The App Manifest is the JSON configuration of an app, created and edited in the [Datadog UI](https://app.datadoghq.com/apps).

`name` : The App’s display name

`description` : A description of what the app does

`icon` : An icon to display alongside the app, specified as a URL.

`description` : A description of what the app does

`main_url` : The URL of the app’s Main Iframe. This Iframe will be mounted on init of the UI, and will persist for the lifetime of the app.

`dev_mode_url`: The URL of the app’s Main Iframe used when [development mode](https://a.cl.ly/NQuJdXep) is enabled, defaults to localhost. Useful to quickly switch between development and production versions of the app.

`features` : A list of the app’s enabled features along with config. The config schema will be different for each feature. See below.

`scopes` : A list of the app’s enabled scopes. See below.

## Main Controller IFrame

All apps must include a **Main Controller IFrame** that successfully interfaces with Datadog through the SDK. The Main Controller IFrame is mounted in the background on every page and is responsible for various administrative actions executed on behalf of the entire application, including storing secrets and managing Datadog authentication credentials. **If the main controller IFrame does not successfully handshake with Datadog, your app may not function properly**.

A minimal controller only need initialize the SDK as follows:

```javascript
// from npm
import { init } from '@datadog/ui-apps-sdk';
// or from injected script:
const { init } = DD_SDK;

init();


```

Although there are no architectural requirements for how to structure your app, you may also wish to use the main controller IFrame to register global event handlers or otherwise coordinate activity amongst the various active features.

## Context Data

**Context** data provides information about the global and specific setting in which features mount in the Datadog UI. Context data will be provided to the SDK in several different ways: 

- When an App IFrame mounts and successfully handshakes with the Datadog UI, it will receive a set of **context** data containing basic information about the setting in which the IFrame renders. This context Data can be accessed with client.getContext() on the SDK client:
```js
client.getContext().then((context) => {
  ...
})
```

- When menu-items (cog menu items, context menu items, or others) are clicked, the SDK in all active iframes will receive a click event. In order to determine where the click event is in the app, click event handlers will receive another set of context data:

```js
client.events.on('dashboard_cog_menu_click', (clickContext) => {

})
```
**Context structure attributes**
- `app`: Global data about the app and user context
    - `name`: The name of the current user
    - `handle`: The current user's email
    - `timeZone`: The current user's time zone. The time zone can differ from the browser's time zone if the user has changed it in the settings.
    - `colorTheme`: The current user's color theme. It can be either `dark` or `light`.
    - `org`: Information about the user's organization
        - `id`: Organization ID
        - `name`: Organization name
        - `features`: A list of the installed app's enabled features.
- `dashboard`: Optional additional data returned when an IFrame or other feature occurs on a dashboard
     - `id`: The ID of the current dashboard
     - `shareToken`: Public dashboard URL, if any
     - `timeframe`: 
          -  `start`: Start time in seconds since unix epoch
          -  `end`: End time in seconds since unix epoch
          -  `isLive`: Whether the dashboard is ‘live’, specifying that widgets should auto-update
     - `templateVars`: Array of dashboard template variables. Items:
         - `name`: Variable name
         - `value`: Variable value
         - `prefix`: Variable prefix
         - `default`: Variable default value
 - `widget`: Optional additional data when an IFrame or other feature occurs in a dashboard widget
     - `id`: Widget ID, if any
     - `definition`: Full widget definition. This includes many fields not fully documented here.
     - `options`: Key-value index of widget options, if any
     - `layout`: Widget layout information
- `menuItem`: Optional information about a menu item when relevant:
    - `key`: The user-provided key of the menu item



 **Events**

Events allow the Datadog UI to communicate with App IFrames, and for App IFrames to communicate with each-other.

 **Standard Events:**

Datadog will send App IFrames events at relevant times in the lifecycle of the application. For example, custom widget iframes will receive a `dashboard_timeframe_change` event when dashboard timeframe changes. Other IFrames will receive events relevant to their use cases, in addition to any global events that may be relevant. These events can be subscribed to with `client.events.on()`:

```js
const unsubscribe = client.events.on('dashboard_timeframe_change', newoptions => {

});
```

 **Custom Events:** 

Apps can listen for and broadcast custom events to each other. An event can be sent to all active app frames with `client.events.broadcast()`:

```js
client.events.broadcast('my_event', myData);
```

All other frames will receive this event and data and can subscribe with client.events.onCustom():

```js
const unsubscribe = client.events.onCustom('my_event', myData => {

});
```


## API Access

Many apps may need to access data from the [Datadog public API](https://docs.datadoghq.com/api/) in order to implement custom functionality. An API client is provided at `client.api` for this purpose:

```js
client.api.get('/api/v1/query', {
  params: {
    from: ...,
    to: ...,
    query: '...'
  }
})
  .then(data => {})
  .catch(e => {})
```
Apps are only able to access API endpoints for which the app has been granted the appropriate Scope. Scope and Data Access in general is a work-in-progress. To start we provide only a single scope option, with more to be added in the near future.

#### Available Scopes:

`metrics_readonly`: Allows read-only access to metrics and query data

### Links & Navigation

Standard anchor links rendered in IFrames will navigate only the iframe window itself, not the entire Datadog app. This may be desirable as a way to switch IFrame content (for example, to change the content of a frame from a list of items to an item detail, and back). However, we expect apps to also want to navigate the main browser window. This can be achieved with `client.location.goTo()`:
```js
client.location.goTo('/infrastructure/map')
```
In addition, apps may open new tabs with `target=”\_\_blank”` links.


# Features

## Dashboard Custom Widget

![custom_widget](https://app.datadoghq.com/static/images/help/app_widgets.gif)

Allows applications to extend the dashboard UI with custom widget items.

#### Configuration:

Cog menu items can be defined statically in the app manifest, or dynamically through the SDK at runtime.

**Static Items**

Custom widgets can be specified as widgets in the feature options as shown in the examples below.

```js
features: [
    ...,
    {
        "name":"dashboard_custom_widget",
        "options":{
        "widgets":[
            {
                // the name of the widget, as it will appear on the tray
                "name":"Your first widget",

                // the key, used to tell apart 2 widgets from the same app
                // don't change this after you created the widget
                "custom_widget_key":"your_first_widget",

                // the code your widget will execute
                "source":"http://localhost:3000",>

                // the icon that will appear in the widget tray
                "icon":"<https://static.datadoghq.com/static/favicon.ico",>

                // configuration options that autogenerate editors
                "options":[
                    {
                        "type":"string",
                        "name":"metric",
                        "label":"Your favorite metric",
                       // renders a dropdown list of options
                        "enum":[
                            "system.cpu.idle",
                            "system.load.1",
                            "system.load.5",
                            "system.load.15"
                        ],
                        "required": true
                    }
                ]
            }
        ]
        }
    }
]
```


**Dynamic Items**

Custom widget items can also be provided dynamically at runtime by the Main Controller Iframe. When a dashboard is loaded, it will send a request to the main controller iframe asking for a set of widget items. Items can be provided by registering a handler with `client.dashboard.customWidget.onRequest`. 

```js
client.dashboard.customWidget.onRequest(() => {
  return {
    widgets: [
      {
        name: "Cheese Widget",
        customWidgetKey: "key1",
        source: "widget",
        options: [
          {
            type: WidgetOptionItemType.STRING,
            name: "favorite-cheese",
            label: "Your favorite cheese",
            enum: ["Chevre", "Gruyere", "Mozzarella"],
 
          },
        ],
        icon: "https://upload.wikimedia.org/wikipedia/en/a/a5/Cheese.png",
      },
    ],
  };
});
```

**Updating Widget Options Dynamically:**

Regardless of whether the widget definition is provided statically in the manifest or dynamically at runtime by the Main Controller Iframe, the widget options can be updated at runtime by the widget Iframe code at anytime.
![options](https://user-images.githubusercontent.com/1262407/116876143-f91ba400-abe9-11eb-83d2-c804c3f9f218.gif)


```js
client.dashboard.customWidget.updateOptions([
  {
    type: WidgetOptionItemType.STRING,
    name: "favorite-cheese",
    label: "Your favorite cheese",
    enum: ["Chevre", "Gruyere", "Mozzarella", "default"],
    // enforce a specific order to display the field in the widget editor
     order: 1
  },
]);
```

**User-friendly Dropdown List Labels:**
You can provider a user-friendly labels for the dropdown list of options using the following syntax:

```js
 enum: [{"label" : "Fancy Cheese Label", "value":"just-plain-cheese"}]
```

**Display order:**

Since widget options can be defined statically in the manifest or dynamically in code, you can enforce a specific display order for your options on the widget editor by providing an optional `order` field with a numerical value.

**Events:**

All IFrames implementing the Custom widgets feature will receive a set of context data relating to the current dashboard and widget, see above for more information. 

```js
const { app, dashboard, widget } = await client.getContext();
```

Additionally, Custom widget iframes can subscribe to the following events:
- `dashboard_custom_widget_options_change`: triggered when the widget is edited. Event handlers will receive an object with the updated configuration options of this widget.
- `dashboard_timeframe_change`: triggered when the timeframe of the current dashboard changes. Event handlers will receive an object with the updated timeframe values.
- `dashboard_template_var_change`: triggered when the template variables of the current dashboard change. Event handlers will receive an object with the updated template variables values.
- `dashboard_cursor_change`: triggered when users mouse over charts on the dashboard. Can be used to track active timestamp across charts.

```js
const unsubscribe = client.events.on('dashboard_custom_widget_options_change', callback);
```

## Side Panels

![sidepanel](https://app.datadoghq.com/static/images/help/app_sidepanels.gif)

Allows applications to open side panels with arbitrary content:

**Configuration:**

Side Panels are defined by a **Side Panel Definition** with the following properties:

- `key` (required) a unique identifier for this side panel
- `source` (required) The relative path to an IFrame to render as the primary side panel content.
- `title` (optional): The title of the side panel. Defaults to the app name.

Panels can be opened from any IFrame with `client.sidePanel.open()` . This method can be called either with a full definition, or with a string **key** referencing a side panel pre-defined in the app manifest

```js
client.sidePanel.open({
    key: "my-panel",
    source: "panel.html"
}, { 
    // optional arguments
    flavor: "cherry",
    drink: "coke"
});
```


**Programmatically closing side panels:**

Side panels may be closed from the sdk with `client.sidePanel.close()`. If called with no argument, any currently active side panel associated with this app will close. If a key is provided as an argument, then only the side panel with the specified key will be closed.

**Events:**

- `side_panel_close`: Broadcasted to all active IFrames when a side panel is closed. Event handlers will receive the full side panel definition of the recently closed side panel.

## Modals
Allows applications to open modal dialogs with arbitrary content:

![modal](https://app.datadoghq.com/static/images/help/app_modals.gif)

**Configuration:**

Modals are defined by a **Modal Definition** with the following properties:

- `key` (required) a unique identifier for this modal
- `title` (optional) A title to render above modal content. Defaults to the app name.
- `message` (optional) A string message to render as main modal content. This is particularly useful for rendering quick confirmation or alert modals not needing custom content.
- `source` (optional) The relative path to an IFrame to render as the primary modal content.
- `size` (optional, `lg`, `md`, or `sm`): The width of the modal
- `action_label` (optional) If provided, a main action button (e.g. “confirm” or “cancel”) will render under the main modal content.
- `action_level` (optional, `primary`, `success`, `danger`, or `warning`): The type of action button to render
- `cancel_label`: (optional) If provided, a cancel button will render below main modal content.

Modals can be opened from any IFrame with `client.modal.open()`.

```js
client.modal.open({
    action_label: 'Yes',
    cancel_label: 'Nevermind',
    title: 'Please Confirm!',
    key: 'confirmation-modal',
    action_level: 'danger',
    message:
        'This modal involves no iframe. It was defined in the manifest and referenced by key.',
    size: 'md'
});
```

**Programmatically closing modals:**
Modals may be closed from the sdk with `client.modal.close()`. If called with no argument, any currently active modal associated with this app will close. If an argument is provided, it will close only a modal with the matching key:

```
client.modal.close('my-modal');
```

**Events:**
- `modal_close`: Broadcasted to all active IFrames when a modal is closed without executing the cancel or main action (e.g. if the user clicks outside or the modal is closed programmatically). Event handlers will receive the full modal definition of the recently closed modal.
- `modal_cancel`: Broadcasted to all active IFrames when the user clicks the **cancel** button. Event handlers will receive the full modal definition of the recently closed modal.
- `modal_action`: Broadcasted to all active IFrames when the user clicks the main action button (e.g. “confirm” or “cancel”). Event handlers will receive the full modal definition of the recently closed modal.

## Dashboard Cog Menu

![dash_ctx_menu](https://app.datadoghq.com/static/images/help/app_cog_menus.png)

Allows applications to extend the dashboard UI with custom cog menu items.

**Configuration:**

Cog menu items can be defined statically in the app manifest, or dynamically through the SDK at runtime.

**Static Items**

Cog menu items can be defined statically in the app manifest. Items defined here will appear in all dashboard cog menus, regardless of context:

```json
features: [
    ...,
    {
        "options": {
            "items": [
                {
                    "action_type": "link",
                    "key": "internal-link",
                    "title": "Internal link item",
                    "href": "/logs"
                },
                {
                    "action_type": "event",
                    "key": "link-item",
                    "title": "Broadcast an event"
                }
            ]
        }
    }
]
```

**Dynamic Items**

Cog menu items can also be provided dynamically at runtime by the Main Controller Iframe. When a cog menu renders on a dashboard, it will send a request to the main controller iframe asking for a set of menu items. Items can be provided by registering a handler with `client.dashboard.cogMenu.onRequest()`. The handler will be provided with a set of context data about the dashboard, so that items may be rendered (or excluded) based on that data:

```javascript

client.dashboard.cogMenu.onRequest(({ dashboard )) => {
  if (dashboard.shareToken) {
  	return {
      items: [{
        actionType: 'event',
        label: 'Share public dashboard',
        key: 'link-item',
        order: 1,

      }]
    }
  }
  
  return {
    items: []
  }
});
```

When a user clicks the item, an event of type `dashboard_cog_menu_context` will be broadcast to all active IFrames.

#### Display order:
Since items can be defined statically in the manifest or dynamically in code, you can enforce a specific display order for your items by providing an optional `order` field with a numerical value.

#### Events:

When cog menu items are clicked, a `dashboard_cog_menu_click` event is broadcast to all active Iframes:

```js
const unsubscribe = client.events.on('dashboard_cog_menu_click', handler);
```

Event handlers will receive a **context** object with information about the active dashboard, and the menu item that was just clicked.

## Widget Context Menu

Allows applications to extend widget visualizations with custom context menu items. At present, custom context menu items are only supported on dashboards.

**Configuration:**

Context menu items should be provided dynamically by the main controller IFrame. When a context menu renders, it will send a request to the main controller iframe asking for a set of menu items. Items can be provided by registering a handler with `client.widgetContextMenu.onRequest()`. The handler will be provided with a set of context data about the widget, so that items may be rendered (or excluded) based on that data:

```javascript

client. widgetContextMenu.onRequest(({ widget )) => {
  if (widget.definition.type === 'timeseries') {
  	return {
      items: [{
        actionType: 'event',
        label: "My custom item",
        key: 'do-my-thing',
      }]
    }
  }
  
  return {
    items: []
  }
});
```

**PLEASE NOTE**: As a best practice, please be sparing in providing context menu items to reduce bloat across the app. Ideally, only provide them when needed 

#### Display order:
Since items can be defined statically in the manifest or dynamically in code, you can enforce a specific display order for your items by providing an optional `order` field with a numerical value.

#### Events:

When cog menu items are clicked, a `widget_context_menu_click` event is broadcast to all active IFrames:

```js
const unsubscribe = client.events.on('widget_context_menu_click', handler);
```

Event handlers will receive a **context** object with information about the widget, and the menu item that was just clicked.


## Authentication
When this feature is enabled, users need be authenticated before using the app.This feature allows you to integrate your existing authentication mechanism like cookie based username/password login with the App Platform. 

![auth](https://app.datadoghq.com/static/images/help/app-auth.gif)

#### How does it work?

[![auth-diagram](https://app.datadoghq.com/static/images/help/app-auth-diagram.jpg)](https://app.datadoghq.com/static/images/help/app-auth-diagram.jpg)

When this feature is enabled, developers need to provide an `authProvider` object programmatically  when initializing the SDK client in the main controller. Example: 

Example:

```js
import { init } from "@datadog/ui-apps-sdk";

const client = init({
  authProvider: {
    url: "https://domain.com/login",
    authStateCallback: async () => {
      try {
        const { username } = await api.getCurrentUser();
        return {
          isAuthenticated: true,
          args: { username },
        };
      } catch (e) {
        return {
          isAuthenticated: false,
        };
      }
    },
  },
});


```

On app load, the app platform will detect the user's auth status based on the logic you define in `authStateCallback` and only render your app if users are authenticated. When users are not authenticated, they will be able to login by clicking the `Authenticate` button. Under the hood, the SDK will open a new tab for you at the provided url. 

By default, we will poll your `authStateCallback` until it either returns `true` or the process times out. This requires no additional effort on your part as a developer if you have an existing authentication flow, however it may result in a short delay between the time the user successfully logs in and when the tab closes. 

If you wish to improve this process, you can manually resolve the authorization flow in one of two ways. In `close` resolution, you can resolve the authorization flow by manually closing the new tab window:

```js
// in your main controller frame
import { init } from "@datadog/ui-apps-sdk";

const client = init({
  authProvider: {
    resolution: 'close',
    ...otherOptions
  },
});
```

```js
// in your login logic, close the window 
loginUser().then(() => {
  window.close();
});
```
Alternately, you can let us know the new authentication state with a helper method distributed in the SDK: 
```js
// in your main controller frame
import { init } from "@datadog/ui-apps-sdk";

const client = init({
  authProvider: {
    resolution: 'message',
    ...otherOptions
  },
});
```

```js
import { resolveAuthFlow } from "@datadog/ui-apps-sdk";

// in your login logic, call the resolve hook:
loginUser().then(() => {
  resolveAuthFlow({
    isAuthenticated: true,
  })'
});
```

Your auth state is now available globally available in your app frames. You can query the current auth state anytime from any frame using `client.auth.getAuthState()` which will return an object with the following format:

```
{
    isAuthenticated: true, // or false,
    args: { username: "matt" } //custom auth related args you want to add to the state. Important: don's store sensitive data here
}

```

**Configuration:**

The Auth Provider is defined with the following properties:

- `url`: (required) The url of your existing login page. This can be an absolute url like `https://domain.com/login` or a relative path to your app like `login`.
- 'authStateCallback':  (required) a callback function indicating whether or not the user is authenticated. It can be async for making API calls or sync for simple checks like reading a value from localStorage or a cookie. For convenience, the return value can also be a simple boolean
```
authStateCallback: () => {
  return getCookieValue("auth_token");
};

```

```
authStateCallback: () => {
  return localStorage.getItem("auth_token");
};

```
- `totalTimeout`: the total period in milliseconds to initiate authentication while the popup is open before it times out. Default is 120000 (2 minutes).

- `requestTimeout `: the time interval in milliseconds before the `authStateCallback()` times out. Default is 20000 (20 seconds).


- `retryInterval`: the time interval in milliseconds to retry initiating authentication while the popup is open. Default is 5000 (5 seconds).

# Shared Formats

## Menu Items

Custom menu items may be provided as part of the `dashboard_cog_menu` and `widget_context_menu` features. In both cases, menu items may be provided in two formats:

**Link items:**

Items of type link can be used to navigate the user within the Datadog UI, or to an external page. They must conform to the following format:

* `key`: A string unique to the item
* `label`: Visible item text
* `type`: "link"
* `href`: Link path. Can be relative to route within Datadog (e.g. /logs), or absolute to link externally (e.g. https://google.com)

**Event items:**

Event items can be used to perform arbitrary actions. When clicked an event will be broadcast to all active IFrames containing context data about the menu item and its environment:

```
client.events.on('<click event>', (context) => {
   ...do stuff
});
```

The exact event name will depend on the feature, please refer to individual documentation. Event items must conform to the following format:

* `key`: A string unique to the item
* `label`: Visible item text
* `type`: "event"
