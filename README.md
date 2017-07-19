# Ember-Orbit [![Build Status](https://secure.travis-ci.org/orbitjs/ember-orbit.png?branch=master)](http://travis-ci.org/orbitjs/ember-orbit) [![Join the chat at https://gitter.im/orbitjs/orbit.js](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/orbitjs/orbit.js?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

Ember-Orbit is a library that integrates
[Orbit.js](https://github.com/orbitjs/orbit) with
[Ember.js](https://github.com/emberjs/ember.js) to provide flexibility and
control in your application's data layer.

Ember-Orbit features:

* Access to the full power of Orbit and its ecosystem, including compatible
  sources, buckets, and coordination strategies.

* A data schema that's declared through simple model definitions.

* Stores that wrap Orbit stores and provide access to their underlying data as
  easy to use records and record arrays. These stores can be forked, edited in
  isolation, and merged back to the original as a coalesced changeset.

* Live-updating filtered record arrays and model relationships.

* The full power of Orbit's composable query expressions.

* The ability to connect any number of sources together with Orbit coordination
  strategies.

* Orbit's git-like deterministic change tracking and history manipulation
  capabilities.

## Relationship to Orbit

Ember-Orbit provides a very thin "Emberified" layer over the top of some core
primitives from Orbit, including `Store`, `Cache`, and `Model` classes that
extend `Ember.Object`. Most common developer interactions with Orbit will be
through these classes.

However, Ember-Orbit does not attempt to wrap _every_ base class from Orbit.
For instance, you'll need to use Orbit's `Coordinator` and coordination
strategies to define relationships between Orbit sources. In this way, you can
install any Orbit `Source` or `Bucket` library and wire them together in your
Ember-Orbit application.

> **Important**: It's strongly recommended that you read the Orbit guides at
[orbitjs.com](http://orbitjs.com) before using Ember-Orbit, since an
understanding of Orbit is vital to making the most of Ember-Orbit.

## Status

Ember-Orbit obeys [semver](http://semver.org) and thus should not be considered
to have a stable API until 1.0. Until then, any breaking changes in APIs or
Orbit dependencies should be accompanied by a minor version bump of Ember-Orbit.

## Demo

[peeps-ember-orbit](https://github.com/cerebris/peeps-ember-orbit) is a simple
contact manager demo that uses Ember-Orbit to illustrate a number of
possible configurations and application patterns.

## Installation

As with any Ember addon, you can install Ember-Orbit in your project with:

```
ember install ember-orbit
```

Ember-Orbit itself already has several Orbit dependencies, including
`@orbit/core`, `@orbit/coordinator`, `@orbit/data`, `@orbit/store`,
`@orbit/utils`, and `@orbit/immutable`.

If you want to use additional Orbit sources and buckets, install them as
regular dependencies. For example:

```
npm install @orbit/jsonapi --save
```

You'll also need to tell Ember-Orbit to include these packages in your build.
Modify your `ember-cli-build.js` file to add them to an `orbit.packages` array
like this:

```
const EmberApp = require('ember-cli/lib/broccoli/ember-app');

module.exports = function(defaults) {
  const app = new EmberApp(defaults, {
    // Orbit-specific options
    orbit: {
      packages: [
        '@orbit/jsonapi',
        '@orbit/indexeddb',
        '@orbit/local-storage',
        '@orbit/indexeddb-bucket',
        '@orbit/local-storage-bucket'
      ]
    }
  });

  return app.toTree();
};
```

> Note: There's nothing particularly Orbit-specific about the implementation
that imports these packages, so expect a more generalized import solution in the
near future.

## Using Ember-Orbit

Ember-Orbit installs the following services by default:

* `dataCoordinator` - An `@orbit/coordinator` `Coordinator` instance that
  manages sources and coordination strategies between them.

* `store` - An Ember-Orbit `Store` instance.

Out-of-the box, you can begin developing using `store` service, which is
injected into every route and controller.

However, the store's contents exist only in memory and will be cleared every
time your app restarts. Let's try adding another source and use the
`dataCoordinator` service to keep it in sync with the store.

### Adding a "backup" source

Sources should be added to the `app/data-sources` directory.

Let's add an IndexedDB source that will serve as a backup. The factory that
generates this source should be defined in `app/data-sources/backup.js` as
follows:

```javascript
import IndexedDBSource from '@orbit/indexeddb';

export default {
  create(injections = {}) {
    injections.name = 'backup';
    injections.namespace = 'my-app';
    return new IndexedDBSource(injections);
  }
};
```

Note that `injections` should include both a `Schema` and a `KeyMap`, which
are created by default for every Ember-Orbit application. We're also adding
a `name` to uniquely identify the source within the coordinator as well as
a `namespace` that will be used to define the IndexedDB database.

Every source that's defined in `app/data-sources` will be discovered
automatically by Ember-Orbit and added to the `dataCoordinator` service.

Next let's define some strategies to synchronize data between sources.

### Defining coordination strategies

Coordination strategies should be added to the `app/data-strategies` directory.

Let's define a strategy to backup changes made to the store into our new
`backup` source. The factory that generates this strategy should be defined
in `app/data-strategies/store-backup.js` as follows:

```javascript
import { SyncStrategy } from '@orbit/coordinator';

export default {
  create() {
    // Backup all store changes (by making this strategy blocking we ensure that
    // the store can't change without the change also being backed up).
    return new SyncStrategy({
      source: 'store',
      target: 'backup',
      blocking: true
    });
  }
};
```

You should also consider adding an event logging strategy to track events
emitted from your sources. Let's add `app/data-strategies/event-logging.js`:

```javascript
import { EventLoggingStrategy } from '@orbit/coordinator';

export default {
  create() {
    return new EventLoggingStrategy();
  }
};
```

A log truncation strategy will keep the size in check by observing the sources
associated with the strategy and truncating their logs when a common transform
has been applied to them all. Let's add `app/data-strategies/log-truncation.js`:

```javascript
import {
  LogTruncationStrategy
} from '@orbit/coordinator';

export default {
  create() {
    return new LogTruncationStrategy();
  }
};
```

### Defining a data bucket

Data buckets should be added to the `app/data-buckets` directory.

Let's define a bucket to store state in IndexedDB. The factory that generates
this strategy should be defined in `app/data-buckets/main.js` as follows:

```javascript
import IndexedDBBucket from '@orbit/indexeddb-bucket';

export default {
  create() {
    return new IndexedDBBucket({ namespace: 'my-app-settings' });
  }
};
```

### Initializing our bucket

We need to ensure that the bucket defined above is injected into all the
sources in our application as well as the `KeyMap` for the application. We can
create a standard Ember initializer to do this - let's say
`app/initializers/orbit.js`:

```javascript
export function initialize(application) {
  application.inject('data-source', 'bucket', 'data-bucket:main');
  application.inject('data-key-map:main', 'bucket', 'data-bucket:main');
}

export default {
  name: 'orbit',
  initialize
};
```

### Activating the coordinator

Next we'll need to activate our coordinator as part of our app's boot process.
The coordinator requires an explicit activation step because the process is
async _and_ we may want to allow developers to do work beforehand.

In our case, we want to restore our store from the backup source before we
enable the coordinator. Let's do this in our application route's `beforeModel`
hook (in `app/routes/application.js`):

```javascript
const { get, inject, Route } = Ember;

export default Route.extend({
  dataCoordinator: inject.service(),

  beforeModel() {
    const coordinator = get(this, 'dataCoordinator');
    const backup = coordinator.getSource('backup');

    return backup.pull(q => q.findRecords())
      .then(transform => this.store.sync(transform))
      .then(() => coordinator.activate());
  }
});
```

This code first pulls all the records from backup and then syncs them
with the main store _before_ activating the coordinator. In this way, the
coordination strategy that backs up the store won't be enabled until after
the restore is complete.

## Contributing to Ember-Orbit

### Installation

* `git clone https://github.com/orbitjs/ember-orbit.git`
* `cd ember-orbit`
* `npm install`

### Running

* `ember serve`
* Visit your app at [http://localhost:4200](http://localhost:4200).

### Running Tests

* `npm test` (Runs `ember try:each` to test your addon against multiple Ember versions)
* `ember test`
* `ember test --server`

### Building

* `ember build`

For more information on using ember-cli, visit [https://ember-cli.com/](https://ember-cli.com/).

## Acknowledgments

Ember-Orbit owes a great deal to [Ember Data](https://github.com/emberjs/data),
which has influenced the design of many of Ember-Orbit's interfaces. Many thanks
to the Ember Data Core Team, including Yehuda Katz, Tom Dale, and Igor Terzic,
for their work.

It is hoped that, by tracking Ember Data's features and interfaces where
possible, Ember-Orbit will also be able to contribute back to Ember Data.


## License

Copyright 2014-2017 Cerebris Corporation. MIT License (see LICENSE for details).
