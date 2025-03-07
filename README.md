[![npm version](https://badge.fury.io/js/@s-group%2Freact-usercentrics.svg)](https://www.npmjs.com/package/@s-group/react-usercentrics) [![Release](https://github.com/s-group-dev/react-usercentrics/actions/workflows/release.yml/badge.svg)](https://github.com/s-group-dev/react-usercentrics/actions/workflows/release.yml)

# `@s-group/react-usercentrics`

React utils for interacting with the [Usercentrics Browser CMP](https://docs.usercentrics.com/#/browser-cmp).

```
npm install --save @s-group/react-usercentrics
```

This code is used in production:

- ✅ https://www.s-kaupat.fi

## Motivation

While the official `@usercentrics/cmp-browser-sdk` npm module is very useful, it is unnecessarily large and would still require some utility functions built around it to be really useful when working in a React application. It has 10 dependencies and an unpacked filesize size of 1.7 MB.

This package aims to be tiny and simple, and has 0 dependencies and an unpacked filesize of 32.7 KB.

## Setup

You will need to set up a Usercentrics service and note down its `settingsId`. You will also need to enable the [Window Event](https://docs.usercentrics.com/#/v2-events?id=usage-as-window-event) in the admin interface, and note down its name (for example, `ucEvent`).

After this you need to render the `UsercentricsScript` component to load the Browser API, and then finally wrap your application in the `UsercentricsProvider`.

```tsx
import { UsercentricsScript, UsercentricsProvider } from '@s-group/react-usercentrics'
import ReactDOM from 'react-dom'
import React from 'react'

import { USERCENTRICS_SETTINGS_ID, USERCENTRICS_EVENT_NAME } from './config'
import MyApplication from './app'

ReactDOM.render(
  <>
    <UsercentricsScript settingsId={USERCENTRICS_SETTINGS_ID} />
    <UsercentricsProvider windowEventName={USERCENTRICS_EVENT_NAME}>
      <MyApplication /** You can interact with Usercentrics inside the provider */ />
    </UsercentricsProvider>
  </>,
  document.body
)
```

### Augmented type-checks for `ServiceId`

Service-specific code needs to target a specific `serviceId`, for example
to check its consent status. You can re-declare a special module to override
the default `string` type of `ServiceId` with an `enum` to support stricter
type-level checks in your application code.

```ts
enum MyServiceId {
  Service1 = 'service-id-1'
  Service2 = 'service-id-2'
}

declare module '@s-group/react-usercentrics/augmented' {
  export type ServiceId = MyServiceId
}
```

or even import the type from another file:

```ts
declare module '@s-group/react-usercentrics/augmented' {
  export type ServiceId = import('../config/usercentrics').ServiceId
}
```

## API

### Types

#### `ConsentType`

When giving consent using the API (instead of customer clicking the Dialog),
consent can be either explicit (e.g. when clicking some custom button) or implicit.

#### `UCWindow`

Augmented window type, possibly including the `UC_UI` API.
Do not declare this globally, but prefer to use the included utility functions instead.

### Components

#### `UsercentricsScript`

The script tag that loads the Usercentrics Browser API.

```tsx
interface UsercentricsScriptProps {
    settingsId: string
    version?: 'production' | 'preview'
}

/** Default production mode */
() => <UsercentricsScript settingsId="1234" />

/** Preview mode for development */
() => <UsercentricsScript settingsId="1234" version="preview" />
```

#### `UsercentricsProvider`

The provider component for listening to events from the Usercentrics Browser API.
Render this once and wrap your application in it.

```tsx
interface UsercentricsProviderProps {
    children: ReactNode
    /**
     * Whether to throw if invalid Service Id has been used.
     * @default false
     */
    strictMode?: boolean
    /**
     * The timeout value in milliseconds after which loading of the Usercentrics
     * script will be assumed to have failed.
     * @default 5000
     */
    timeout?: number
    /**
     * The configured window event name from Usercentrics admin interface.
     * @see https://docs.usercentrics.com/#/v2-events?id=usage-as-window-event
     */
    windowEventName: string
}

/** Basic usage */
() => (
  <UsercentricsProvider windowEventName="ucEvent">
    <App />
  </UsercentricsProvider>
)

/** Custom timeout in milliseconds */
() => (
  <UsercentricsProvider timeout={100} windowEventName="ucEvent">
    <App />
  </UsercentricsProvider>
)
```

#### `UsercentricsDialogToggle`

A button component for opening the Usercentrics dialog where users can
adjust their consent settings, and read info about services.

The button will be disabled until Usercentrics has been initialized,
so for example ad-blockers might prevent it from ever activating.

```tsx
interface UsercentricsDialogToggleProps extends ButtonHTMLAttributes<HTMLButtonElement> {
    children: ReactNode
}

/** Basic usage */
() => <UsercentricsDialogToggle>Manage my consents</UsercentricsDialogToggle>
```

### Hooks

#### `useHasServiceConsent`

Whether the specific Usercentrics service has been given consent.
Returns `true` or `false` based on consent status, or `null` when unknown (not yet loaded).

 **Warning:** it's best to assume no consent until this hook returns `true`

```tsx
() => {
  const hasConsent = useHasServiceConsent('my-service-id')

  useEffect(() => {
    if (hasConsent) {
      loadMyService()
    }
  }, [hasConsent])
}
```

#### `useIsFailed`

Returns `true` if Usercentrics failed to load inside the
timeout configured in `UsercentricsProvider`. This means
consent status is unknown and will default to `false`,
so no services can be used.

```tsx
() => {
  const isFailed = useIsFailed()

  useEffect(() => {
    if (isFailed) {
      console.error('Failed to load consents! This site might not work properly.')
    }
  }, [isFailed])
}
```

#### `useIsInitialized`

Returns `true` if Usercentrics has been initialized and consents can be given.

```tsx
() => {
  const isInitialized = useIsInitialized()

  useEffect(() => {
    if (isInitialized) {
      console.info('Usercentrics has initialized!')
    }
  }, [isInitialized])
}
```

#### `useIsOpen`

Returns `true` if the Usercentrics dialog is currently open on the page.

```tsx
() => {
  const isOpen = useIsOpen()

  useEffect(() => {
    if (isOpen) {
      console.debug('Usercentrics dialog is open and probably covers the whole screen')
    } else {
      console.debug('Usercentrics dialog is now closed')
    }
  }, [isOpen])
}
```

#### `useServiceInfo`

Returns basic info for specific Usercentrics service, or null if not found.

The typing is *not complete* and contains only the info used internally:

- `id` of the service, autogenerated by Usercentrics
- `name` of the service, configured in the admin interface
- `consent.status` of the service

See also https://docs.usercentrics.com/#/cmp-v2-ui-api?id=getservicesbaseinfo

```tsx
() => {
  const serviceInfo = useServiceInfo('my-service-id')

  useEffect(() => {
    if (!serviceInfo) {
      console.error('Service not found for "my-service-id"')
    } else {
      console.info(`Consent for ${serviceInfo.name}: ${serviceInfo.consent.status}`)
    }
  }, [serviceInfo])
}
```

#### `useServiceFullInfo`

Returns full info for specific Usercentrics service, or null if not found.
This triggers an extra API call and also returns `null` while loading.

The typing is *not complete* and contains only the info used internally:

- `id` of the service, autogenerated by Usercentrics
- `name` of the service, configured in the admin interface
- `consent.status` of the service
- `description` text of the service

See also https://docs.usercentrics.com/#/cmp-v2-ui-api?id=getservicesfullinfo

```tsx
() => {
  const serviceInfo = useServiceFullInfo('my-service-id')

  useEffect(() => {
    if (serviceInfo) {
      console.info(`The ${serviceInfo.name} service is used to ${serviceInfo.description}`)
    }
  }, [serviceInfo])
}
```

### Utils

#### `showFirstLayer`

Programmatic way to show First Layer.

See also https://docs.usercentrics.com/#/cmp-v2-ui-api?id=showfirstlayer

```tsx
showFirstLayer()
```

#### `showSecondLayer`

Programmatic way to show Second Layer. If a service/vendor Id value is passed,
Second Layer will open the right tab, scroll to the given service/vendor entry and expand it.
If no Id is passed, Second Layer will be shown without srcolling to any specific service/vendor.

See also https://docs.usercentrics.com/#/cmp-v2-ui-api?id=showsecondlayer

```tsx
showSecondLayer()
```

```tsx
showSecondLayer('my-service-id')
```

#### `getServicesBaseInfo`

A method to get array of all services with their basic information

See also https://docs.usercentrics.com/#/cmp-v2-ui-api?id=getservicesbaseinfo

```tsx
const services = getServicesBaseInfo()

const myService = services.find((service) => service.id === 'my-service-id')
```

#### `getServicesFullInfo`

A method to get array of all services with their full information.
An extra api request will be made, therefore the return represents
the eventual completion (or failure) of an asynchronous operation
and its returning value.

See also https://docs.usercentrics.com/#/cmp-v2-ui-api?id=getservicesfullinfo

```tsx
const services = await getServicesFullInfo()

const myService = services.find((service) => service.id === 'my-service-id')
```

#### `hasServiceConsent`

Returns true if Usercentrics service has been given consent

```tsx
const services = getServicesBaseInfo()
const myService = services.find((service) => service.id === 'my-service-id')
const hasConsent = hasServiceConsent(myService)

if (hasConsent) {
  loadMyService()
}
```

#### `acceptService`

A method for accepting a single service.

See also https://docs.usercentrics.com/#/cmp-v2-ui-api?id=acceptservice

```tsx
await acceptService('my-service-id')
```

```tsx
await acceptService('my-service-id', ConsentType.Explicit)
```

```tsx
await acceptService('my-service-id', ConsentType.Implicit)
```
