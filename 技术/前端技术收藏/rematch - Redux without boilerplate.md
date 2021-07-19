rematch - Redux without boilerplate

[![Chat on slack](../../_resources/eabbdbfc26a144d897bae3fc0aa49576.svg)](https://rematchjs.slack.com/) ![Rematch CI](../../_resources/1f63f6dae4bf425ea6ad1a20a8664e21.svg)![npm (tag)](../../_resources/c97c97483a9f42ac80b6c5f1588d90f9.svg)[![Bundle size](../../_resources/ecc726b5d09d47208308f82abc05518a.svg)](https://img.shields.io/badge/bundlesize-~5kb-brightgreen.svg?style=flat-square)[![File size](../../_resources/88d8377e2f3148fa936a24516e3bef5f.svg)](https://img.shields.io/badge/dependencies-redux-brightgreen.svg?style=flat-square)[![lerna](../../_resources/72c41621af2844d9ae9a73bc8e8ad38d.svg)](https://lerna.js.org/)

> Rematch is Redux best practices without the boilerplate. No more action types, action creators, switch statements or thunks.

## [Features](#/introduction?id=features)

Redux is an amazing state management tool, supported by a healthy middleware ecosystem and excellent devtools. Rematch builds upon Redux by reducing boilerplate and enforcing best practices. It provides the following features:

- No configuration needed
- Reduces Redux boilerplate
- Built-in side-effects support
- [React Devtools](https://github.com/facebook/react/tree/master/packages/react-devtools) support
- TypeScript support
- Supports dynamically adding reducers
- Supports hot-reloading
- Allows to create multiple stores
- Supports React Native
- Extendable with plugins
- Many plugins available out of the box:
    - for persisting data with [redux-persist](https://github.com/rt2zz/redux-persist)
    - for wrapping state with [immer.js](https://github.com/immerjs/immer)
    - for creating selectors with [reselect](https://github.com/reduxjs/reselect)
    - …and others

## [Redux vs Rematch](#/introduction?id=redux-vs-rematch)

|     | Redux | Rematch |
| --- | --- | --- |
| simple setup ‎ |     | ‎✔  |
| less boilerplate |     | ‎✔  |
| readability |     | ‎✔  |
| configurable | ‎ ✔ | ‎✔  |
| redux devtools | ‎✔  | ‎✔  |
| generated action creators | ‎   | ‎✔  |
| async | thunks | ‎async/await |

## [How to start](#/introduction?id=how-to-start)

[Learn how to integrate easily on Javascript or Typescript (React, Vue…)](#/quick-start)

## [Migrate From Redux](#/introduction?id=migrate-from-redux)

Migrating from Redux to Rematch may only involve minor changes to your state management, and no necessary changes to your view logic. See the [migration reference](#/migration-guide) for the details.

## [Composable Plugins](#/introduction?id=composable-plugins)

Rematch and its internals are all built upon a plugin pipeline. As a result, developers can make complex custom plugins that modify the setup or add data models, often without requiring any changes to Rematch itself. See the [plugins](#/plugins/summary) developed by the Rematch team or the [API for creating plugins](#/api/plugins?id=plugins-api).

## [All plugins](#/introduction?id=all-plugins)

### [Deprecated](#/introduction?id=deprecated)

- Create a [GitHub issue](https://github.com/rematch/rematch/issues) for bug reports, feature requests, or questions
- Add a ⭐️ [star on GitHub](https://github.com/rematch/rematch) to support the project!

## [License](#/introduction?id=license)

This project is licensed under the [MIT license](https://github.com/rematch/rematch/blob/master/LICENSE).