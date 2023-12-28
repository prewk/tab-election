# Tab Election

Provides leadership election and communication in the browser across tabs *and* workers using the Locks API and BroadcastChannel. It works in modern browsers.

The [Locks API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Locks_API) allows us to have a very reliable leadership election, with virtually no delay in database or server connections and app startup time. When the existing leader is closed, the next tab will become the new leader immediately. The Tab interface allows calls and messages to be queued before a leader is elected and sent afterwards. The Tab interface supports everything you need to have all tabs communicate with one leader for loading, saving, and syncing data between tabs, including calling API methods the leader provides, broadcasting messages to other tabs, and state syncing.

## Install

```
npm install --save tab-election
```

## API

```js
import { Tab } from 'tab-election';

const tab = new Tab();

tab.waitForLeadership(() => {
  // establish websocket, database connection, or whatever is needed as the leader
});
```

If a tab needs to stop being a leader (or waiting to become one) you can call close on the returned elector and allow garbage collection.

```js
import { Tab } from 'tab-election';

const tab = new Tab('namespace');

tab.waitForLeadership(() => {
  // establish websocket, database connection, or whatever is needed as the leader, return an API
  return {
    async loadData() {
      // return await db.load(...);
    }
  }
});

// ... sometime later, perhaps a tab is stale or goes into another state that doesn't need/want leadership
tab.close();
```

To communicate between tabs, send and receive messages.

```js
import { Tab } from 'tab-election';

const tab = new Tab('namespace');

tab.addEventListener('message', event => console.log(event.data));
tab.send('This is a test'); // will not send to self, only to other tabs
```

To keep state (any important data) between the current leader and the other tabs, use `state()`. Use this to let the
other tabs know when the leader is syncing, whether it is online, or if any errors have occured. `state()` will return
the current state of the leader and `state(data)` will set the current state if the tab is the current leader.

The state object can contain anything that is supported by the [Structured Clone Algorithm](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm)
including Dates, RegExes, Sets, and Maps.

```js
import { Tab } from 'tab-election';

const tab = new Tab('namespace');

tab.waitForLeadership(() => {
  // establish websocket, database connection, or whatever is needed as the leader
  tab.setState({ connected: false });
  // connect to the server ...
  tab.setState({ connected: true });
});

tab.addEventListener('state', event => console.log('The leader is connected to the server?', event.data.connected));
```

To allow tabs to call methods on the leader (including the leader), use the `call()` method. The return result is always
asyncronous. The API that is callable should be returned from the `waitForLeadership` callback. If the leader has
established a connection to the server and/or database, this may be used for other tabs to get/save data through that
single connection.

```js
import { Tab } from 'tab-election';

const tab = new Tab('namespace');

tab.waitForLeadership(async () => {
  // Can have async instructions here. Calls to `call` in any tab will be queued until the API is returned.
  const db = await connectToTheDatabase();

  return {
    saveData(data) {
      // ...
      return true;
    }
  }
});

const result = await tab.call('saveData', { myData: 'foobar' });
if (result === true) {
  console.log('Successfully saved');
}
```

If a tab wants to make calls to the leader, send and receive messages, and know the state, but it does not want to ever
become the leader, then don't call `waitForLeadership`. This is useful when workers are used for leadership and UI
contexts make the requests and display state.

```js
import { Tab } from 'tab-election';

const tab = new Tab('namespace');

const result = await tab.call('saveData', { myData: 'foobar' });
if (result === true) {
  console.log('Successfully saved');
}
```
