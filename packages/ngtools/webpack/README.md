# Angular Compiler Webpack Plugin

Webpack 5.x plugin for the Angular Ahead-of-Time compiler. The plugin also supports Angular JIT mode.
When this plugin is used outside of the Angular CLI, the Ivy linker will also be needed to support
the usage of Angular libraries. An example configuration of the Babel-based Ivy linker is provided
in the linker section. For additional information regarding the linker, please see: https://v13.angular.io/guide/creating-libraries#consuming-partial-ivy-code-outside-the-angular-cli

## Usage

In your webpack config, add the following plugin and loader.

```typescript
import { AngularWebpackPlugin } from '@ngtools/webpack';

exports = {
  /* ... */
  module: {
    rules: [
      /* ... */
      {
        test: /\.[jt]sx?$/,
        loader: '@ngtools/webpack',
      },
    ],
  },

  plugins: [
    new AngularWebpackPlugin({
      tsconfig: 'path/to/tsconfig.json',
      // ... other options as needed
    },
    [
      // list of typescript transformers
    ]),
  ],
};
```

The loader works with webpack plugin to compile the application's TypeScript. It is important to include both, and to not include any other TypeScript loader.
To use typsescript transformers add them in as second argument. !WARNING!: I guess this isn't supported for a reason, so make sure your output is correct.

### Transformer Example
Example of removing statements in typescript.
```typescript
import ts = require("typescript");

function transformer(
  program?: ts.BuilderProgram // <-- Seems like this can supply you important functions like typechecker, but not required to be input
): ts.TransformerFactory<ts.SourceFile> {
    return (context: ts.TransformationContext) => {
        const visitNode: ts.Visitor = (node: any) => {
            if (node.expression &&
                ts.isCallExpression(node.expression) &&
                ts.isIdentifier(node.expression.expression)) {
                const decoratorIdentifier = node.expression.expression;
                // remove this node as its just meant for metadata, not for compiling
                if (decoratorIdentifier.text == "MyMetaDataDecorator") {
                        return undefined;
                }
            }
            return ts.visitEachChild(node, visitNode, context);
        };

        return (sourceFile: ts.SourceFile) => {
            let updatedSourceFile = ts.visitEachChild(sourceFile, visitNode, context);
            return updatedSourceFile;
        };
    };
}

// create plugin
new AngularWebpackPlugin({
    tsconfig: 'path/to/tsconfig.json',
    // ... other options as needed
  },
  [
    transformer
  ]
)
```

## Options

- `tsconfig` [default: `tsconfig.json`] - The path to the application's TypeScript Configuration file. In the `tsconfig.json`, you can pass options to the Angular Compiler with `angularCompilerOptions`. Relative paths will be resolved from the Webpack compilation's context.
- `compilerOptions` [default: none] - Overrides options in the application's TypeScript Configuration file (`tsconfig.json`).
- `jitMode` [default: `false`] - Enables JIT compilation and do not refactor the code to bootstrap. This replaces `templateUrl: "string"` with `template: require("string")` (and similar for styles) to allow for webpack to properly link the resources.
- `directTemplateLoading` [default: `true`] - Causes the plugin to load component templates (HTML) directly from the filesystem. This is more efficient if only using the `raw-loader` to load component templates. Do not enable this option if additional loaders are configured for component templates.
- `fileReplacements` [default: none] - Allows replacing TypeScript files with other TypeScript files in the build. This option acts on fully resolved file paths.
- `inlineStyleFileExtension` [default: none] - When set inline component styles will be processed by Webpack as files with the provided extension.

## Ivy Linker

The Ivy linker can be setup by using the Webpack `babel-loader` package.
If not already installed, add the `babel-loader` package using your project's package manager.
Then in your webpack config, add the `babel-loader` with the following configuration.
If the `babel-loader` is already present in your configuration, the linker plugin can be added to
the existing loader configuration as well.
Enabling caching for the `babel-loader` is recommended to avoid reprocessing libraries on
every build.
For additional information regarding the `babel-loader` package, please see: https://github.com/babel/babel-loader/tree/main#readme

```typescript
import linkerPlugin from '@angular/compiler-cli/linker/babel';

exports = {
  /* ... */
  module: {
    rules: [
      /* ... */
      {
        test: /\.[cm]?js$/,
        use: {
          loader: 'babel-loader',
          options: {
            cacheDirectory: true,
            compact: false,
            plugins: [linkerPlugin],
          },
        },
      },
    ],
  },
};
```
