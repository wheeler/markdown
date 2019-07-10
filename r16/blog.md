# Everything we had to do to upgrade to React 16

-or-

a stroll through dependency heck

## High Level

Upgrading to React 16 at its highest level required a few major dependency changes and resolution of a few breaking changes. Here is a summary of the most major ones:

- Upgrade to `enzyme@3`

  We have lots of Enzyme tests for our frontend that we want to keep - but Enzyme 2 doesn't work with React 16.

- Create a permanent react root for our Single Page App

  Because of the history of how it was built there was no single permanent react root component in our single page app. We had to restructure so that there was so that we could do the next item.

- Upgrade `react-hot-loader@4`

  We use `react-hot-loader` to speed the development experience with "hot" component updating as we code and re-save components. The version we were using wouldn't support React 16 so we would need to upgrade to v4 to get full React 16 support. v4 uses a `hot` HOC to establish the hot loading root rather than using webpack to define hot entry points. We need a root component to `hot` HOC which is the reason for the previous item. We also needed to re-structure the import tree a bit for the new version of hot loader to always find its hot entry point at the root.

- At this point we started to actually try a branch with React 16 to find additional dependencies and problems

- Upgrade to chai@4, sinon@7, mocha@6

  We started to have `performance.mark is not a function` errors started happening in some tests with sinon timers in React 16. This was a `sinon/lolax` issue which has been resolved in sinon 7.1.0. See sinonjs/lolex#136 and sinonjs/lolex#160. There was a workaround proposed in some issue comments but it broke other tests for us. So, upgrading sinon was the better choice. Upgrading to sinon@7 required upgrading chai@4.

  Upgrading mocha@6 was a red-herring to try to fix what seemed like an inability to actually trap rendering errors with React 16 error boundaries. It wasn't too hard and didn't waste too much time, but it did not fix that problem - the tests and some of our helpers needed to be re-written.

- Upgrade other minor packages

  There were other packages that had to be upgraded mostly for simple version compatibility listings, without breaking changes or with very minor breaking changes.

- Grind out React breaking changes and deprecations.

- Find other unfortunate interactions and fix them.

## Details
