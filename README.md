# play-angular-cli


## Prerequisites

* Install activator and sbt
* Install node, npm and angular-cli

## Goal

![alt tag](webpack.png)

## Setup

### PlayFramework backend

```
$ activator new backend play-scala
```

## Configuration

`replace` the source code in controllers.HomeController.scala by  

```scala
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

```yaml

<...>

# Bundle files generated by Webpack
GET     /dist/*file                 controllers.HomeController.dist(file)
```

`add` the below source code in `built.sbt`

```sbt

<...>

/* ================================= WEBPACK ================================== */

val frontEndProjectName = "frontend"
val backEndProjectName = "backend"
val distDirectory = ".." + backEndProjectName + "public/dist"

// Starts: angularCLI build task
val frontendDirectory = baseDirectory {_ /".."/frontEndProjectName}

val webpack = sys.props("os.name").toLowerCase match {
  case os if os.contains("win") ⇒ "cmd /c node_modules\\.bin\\webpack"
  case _ ⇒ "node_modules/.bin/webpack"
}

val params = " --config webpack.config.js --progress --colors --display-error-details"

val ngBuild = taskKey[Unit]("webpack build task.")
ngBuild := { Process( webpack + params , frontendDirectory.value) ! }
(packageBin in Universal) <<= (packageBin in Universal) dependsOn ngBuild
// Ends.

```

# Frontend

### angular-cli frontend
```bash
$ ng new frontend
```


## Adding webpack
```bash
$ cd frontend
```
## Create webpack configuration file   

> In ther frontend directory

* `copy`: `config/vendor.ts` to `src`
```
$ cp ../config/vendor.ts src/
```

* create file `webpack.config.js`

* copy from webpack-template
```
$ cp ../config/webpack.config.js .
```

* run install
```
$ node_modules/.bin/webpack --config webpack.config.js
```


## webpack [loaders](https://webpack.js.org/concepts/loaders/)

* Install [TypeScript's loaders](https://webpack.js.org/guides/webpack-and-typescript/)
```bash
$ npm install ng-router-loader awesome-typescript-loader angular2-template-loader --save-dev 
```

Style loaders
```bash
$ npm install to-string-loader style-loader css-loader --save-dev
```

Asset Copy plugin
```bash
$ npm install copy-webpack-plugin --save-dev
```

Extra Libraries (as needed)
```bash
$  npm install lodash moment include-media bootstrap --save
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
