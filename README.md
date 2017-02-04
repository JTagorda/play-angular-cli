# play-angular-cli


## Prerequisites

* Install activator and sbt
* Install node, npm and angular-cli

## Setup

### PlayFramework backend

```
$ activator new backend play-scala
```

## Configuration

`replace` the source code in controllers.HomeController.scala by  

```
package controllers

import javax.inject.{Inject,Singleton}

import play.api.{Environment,Mode}
import play.api.mvc.{Controller,Action}

// Import required for injection
import play.api.libs.ws.WSClient
import scala.concurrent.ExecutionContext

/**
 * This controller creates an `Action` to handle HTTP requests to the
 * application's home page.
 */
@Singleton
class HomeController @Inject()(ws: WSClient, environment: Environment)(implicit ec: ExecutionContext) extends Controller {

  def index = Assets.versioned(path="/public/dist", "index.html")

  def dist(file: String) = environment.mode match {
    case Mode.Dev => Action.async {
      ws.url("http://localhost:4200/dist/" + file).get().map { response =>
        Ok(response.body)
      }
    }
    // If Production, use build files.
    case Mode.Prod => Assets.versioned(path="/public/dist", file)
  }

}
```

`add` the below source code to `conf/routes`   

```

<...>

# Bundle files generated by Webpack
GET     /dist/*file                 controllers.HomeController.dist(file)
```

`add` the below source code in `built.sbt`

```

<...>

/* ================================= WEBPACK ================================== */

val frontEndProjectName = "frontend"
val backEndProjectName = "backend"
val distDirectory = ".." + backEndProjectName + "public/dist"


// Starts: angularCLI build task
val frontendDirectory = baseDirectory {_ /".."/frontEndProjectName}

val webpack = "webpack --progress --colors --display-error-details"

val ngBuild = taskKey[Unit]("webpack build task.")
ngBuild := { Process(webpack , frontendDirectory.value) ! }
(packageBin in Universal) <<= (packageBin in Universal) dependsOn ngBuild
// Ends.


// Starts: ngServe process when running locally and build actions for production bundle
PlayKeys.playRunHooks <+= frontendDirectory.map(base => ng(base))
// Ends.
```

`create` a file with the below source code in `project/ng.scala`
```
import java.io.File
import java.net.InetSocketAddress

import play.sbt.PlayRunHook
import sbt.Process

object ng {
    def apply(base: File): PlayRunHook = {

        object ngServe extends PlayRunHook {

            var process: Option[Process] = None // This is really ugly, how can I do this functionally?

            override def afterStarted(addr: InetSocketAddress): Unit = {
                process = Some (Process( "ng serve --watch " , base).run)
            }

            override def afterStopped(): Unit = {
                process.foreach(_.destroy)
                process = None
            }
        }

        ngServe
    }
}
```

# Frontend

### angular-cli frontend
```
$ ng new frontend
```


## Adding webpack
```
$ cd frontend
```

* Add webpack to the project (package.json)  
  `note:` that using the parameters --save-dev and -save will respectively 
  add the packages to package.json sections devDependencies and dependencies

  `warning:` do not install webpack it's already installed by angular-cli
  > npm install webpack@beta --save-dev 


## webpack [loaders](https://webpack.js.org/concepts/loaders/)

* Install [TypeScript's loaders](https://webpack.js.org/guides/webpack-and-typescript/)
  I use ts-loader (instead of awesome-typescript-loader) and angular2-template-loader 
```
$ npm install ts-loader angular2-template-loader --save-dev 
```

Style loaders
```
$ npm install style-loader raw-loader --save-dev
```


* Create webpack configuration file   
   `rename`: `'/.index.js'` to `'./src/main.ts'` in `entry:`  
   `rename`: `'/'` to `'../backend/public/dist'` in `output.path:`  
   `add`: `/dist` to `output.publicPath:`

* Copying file `vendor.ts`
`copy`: `webpack-template/vendor.ts` to `src`

* create file `webpack.config.js`

webpack.config.js
```
const HtmlWebpackPlugin        = require('html-webpack-plugin'); //installed via npm

const path                     = require('path');
const CommonsChunkPlugin       = require('webpack/lib/optimize/CommonsChunkPlugin');
const ContextReplacementPlugin = require('webpack/lib/ContextReplacementPlugin');
const DefinePlugin             = require('webpack/lib/DefinePlugin');

const ENV  = process.env.NODE_ENV = 'development';

const metadata = {
  env : ENV
};

module.exports = {
  devtool: 'source-map',
  entry: {
    'main'  : './src/main.ts',
    'vendor': './src/vendor.ts'
  },
  module: {
    loaders: [
      {test: /\.css$/,  loader: 'raw-loader', exclude: /node_modules/},
      {test: /\.css$/,  loader: 'style!css?-minimize', exclude: /src/},
      {test: /\.html$/, loader: 'raw-loader'},
      {test: /\.ts$/,   loaders: [
          {loader: 'ts-loader', query: {compilerOptions: {noEmit: false}}},
          {loader: 'angular2-template-loader'}
        ],
        exclude: [/\.(spec|e2e)\.ts$/]
      }
    ]
  },
  output: {
    filename: '[name].js',
    path: '../backend/public/dist',
    publicPath: "/dist"
  },
  plugins: [
    new CommonsChunkPlugin({name: 'vendor', filename: 'vendor.bundle.js', minChunks: Infinity}),
    new DefinePlugin({'webpack': {'ENV': JSON.stringify(metadata.env)}}),
    new ContextReplacementPlugin(
      // needed as a workaround for the Angular's internal use System.import()
      // The (\\|\/) piece accounts for path separators in *nix and Windows
      /angular(\\|\/)core(\\|\/)(esm(\\|\/)src|src)(\\|\/)linker/,
      path.join(__dirname, 'src') // location of your src
    ),
    new HtmlWebpackPlugin({template: './src/index.html'})
  ],
  resolve: {
    extensions: ['.ts', '.js']
  }
};
```
# Route Module Management

```
$ npm install angular2-router-loader --save-dev 
```
Add the module entries
```
  entry: {
    'admin'  : './src/app/admin/admin.module.ts',
    'crisis-center'  : './src/app/crisis-center/crisis-center.module.ts',
    <...>
  },
```
and 


## webpack plugins

* Add the [html-webpack-plugin](https://webpack.js.org/concepts/plugins/#configuration) to change the index.html file  
  this change will allow `<script src=/asset/...>`

```
$ npm install html-webpack-plugin --save-dev
```
* Add HtmlWebpackPlugin object

webpack.config.js
```
const HtmlWebpackPlugin = require('html-webpack-plugin'); //installed via npm
```
* point the template to the index.html file  

```
  plugins: [
    new HtmlWebpackPlugin({template: './src/index.html'}),
    ...
  ]
```



## Running (Prod)

```
$ sbt dist
```

```
$ sbt playGenerateSecret
$ export PLAY_APP_SECRET="put the result here"
```

```
$ unzip target/universal/backend-1.0-SNAPSHOT.zip
$ backend-1.0-SNAPSHOT/bin/backend -Dplay.crypto.secret=$PLAY_APP_SECRET
```




# Reference

heavily inspired by https://github.com/wigahluk/play-webpack  

## angular.io and webpack guide

https://angular.io/docs/ts/latest/guide/webpack.html
