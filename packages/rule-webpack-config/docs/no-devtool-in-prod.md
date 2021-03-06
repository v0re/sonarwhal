# Check `devtool` property in webpack configuration (`no-devtool-in-prod`)

## Why is this important?

The `devtool` property significantly increases the size of your JavaScript
bundles.

## What does the rule check?

This checks if you are using the proper `devtool` property for production
webpack builds.

### Example that **trigger** the rule

```js
module.exports = {
    devtool: 'eval',
    entry: ['entry'],
    output: {
        filename: 'bundle.js',
        path: path.join(__dirname, 'dist')
    }
};
```

### Examples that **pass** the rule

```js
module.exports = {
    entry: ['entry'],
    output: {
        filename: 'bundle.js',
        path: path.join(__dirname, 'dist')
    }
};
```

## Further Reading

* [Webpack Documentation][webpack docs]
* [devtool property docs][devtool docs]

[webpack docs]: https://webpack.js.org/concepts/
[devtool docs]: https://webpack.js.org/configuration/devtool
