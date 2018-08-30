---
theme : "night"
transition: "slide"
highlightTheme: "dracula"
---
## More About Go Modules
Francesco Farina - 
30/08/2018
<p><span id="date"></span></p>
Credits to [Roberto Selbach](https://roberto.selbach.ca/intro-to-go-modules/)

---

### Modules and GOPATH
 - `go mod` outside of `GOPATH`
 - `GO111MODULE=on` inside of `GOPATH`

---

### Creating a Module
```shell
$ mkdir testmod
$ cd testmod
```
Simple package
```go
package testmod

import "fmt"

// Hi returns a friendly greeting
func Hi(name string) string {
   return fmt.Sprintf("Hi, %s", name)
}
```
Switch module on
```shell
$ go mod init github.com/robteix/testmod
go: creating new go.mod: module github.com/robteix/testmod
```

--

This creates the _go.mod_ file
```
module github.com/robteix/testmod
```
Push it to a repo
```
$ git init 
$ git add * 
$ git commit -am "First commit" 
$ git push -u origin master
```
Usage before modules
```
$ go get github.com/robteix/testmod
```
<small>This would fetch the latest code in master! 😰</small>

---

### Module Versioning

 - Semantic Versioning
 - Repo Tags
  - fetch the _latest tagged_ version

--

### First release
 - Tag the repo with the version number
```shell
$ git tag v1.0.0
$ git push --tags
```
 - Might be a good idea to
```shell
$ git checkout -b v1
$ git push -u origin v1
```
<small>Go doesn’t enforce that, but might be a good idea</small>

--

### Module Usage
```go
package main

import (
 	"fmt"

 	"github.com/robteix/testmod"
)

func main() {
 	fmt.Println(testmod.Hi("user"))
}
```
Instead of

`go get github.com/robteix/testmod`

```shell
$ go mod init usermod
```
```
module usermod
```

--

### Module Usage
```shell
$ go build
go: finding github.com/robteix/testmod v1.0.0
go: downloading github.com/robteix/testmod v1.0.0
```
```
module usermod
require github.com/robteix/testmod v1.0.0
```

---

### Bugfix release
```go
package testmod

import "fmt"

// Hi returns a friendly greeting
func Hi(name string) string {
-       return fmt.Sprintf("Hi, %s", name)
+       return fmt.Sprintf("Hi, %s!", name)
}
```

```shell
$ git commit -m "Emphasize our friendliness" testmod.go
$ git tag v1.0.1
$ git push --tags origin v1
```

---

### Updating modules

 - `go get -u` <small>use the latest minor or patch releases (i.e. 1.0.0 -> 1.0.1 or, if available, 1.1.0)</small>
 - `go get -u=patch` <small>use the latest patch releases (i.e., 1.0.0 -> 1.0.1 but not to 1.1.0)</small>
 - `go get package@version` <small>update to a specific version (say, github.com/robteix/testmod@v1.0.1)</small>

--

### Updating modules

`go.mod` file after running `go get -u`
```
module usermod
require github.com/robteix/testmod v1.0.1
```

---

### Major versions
From the point of view of Go modules, a major version is a completely different package

<p class="fragment">
    <small>Two versions of a library that are not compatible with each other are two different libraries</small>
</p>

--

### Major versions

```go
package testmod

import (
 	"errors"
 	"fmt" 
) 

// Hi returns a friendly greeting in language lang
func Hi(name, lang string) (string, error) {
 	switch lang {
 	case "en":
 		return fmt.Sprintf("Hi, %s!", name), nil
 	case "pt":
 		return fmt.Sprintf("Oi, %s!", name), nil
 	case "es":
 		return fmt.Sprintf("¡Hola, %s!", name), nil
 	case "fr":
 		return fmt.Sprintf("Bonjour, %s!", name), nil
 	default:
 		return "", errors.New("unknown language")
 	}
}
```

<span class="fragment"><small>Time to bump up the version to 2.0.0</small></span>

--

### Major versions

__Versions 2 and over should change the import path. They are different libraries now.__

<span class="fragment"><small>Appending a new version path to the end of our module name</small></span>
<span class="fragment"><small>`module github.com/robteix/testmod/v2`</small></span>

--

### Major versions

Releasing a major

```shell
$ git commit testmod.go -m "Change Hi to allow multilang"
$ git checkout -b v2 # optional but recommended
$ echo "module github.com/robteix/testmod/v2" > go.mod
$ git commit go.mod -m "Bump version to v2"
$ git tag v2.0.0
$ git push --tags origin v2 # or master if we don't have a branch
```

---

### Upgrading to a major version

 - Existing software _will not break_
  - `go get -u` will not get version `2.0.0`

--

### Upgrading to a major version

Let's uprade

```go
package main

import (
    "fmt"
    "github.com/robteix/testmod/v2" 
)

func main() {
    g, err := testmod.Hi("Roberto", "pt")
    if err != nil {
        panic(err)
    }
    fmt.Println(g)
}
```
<small>
- `go build` will fetch `v2.0.0`
- Go refers to the module by its proper name `testmod`
 - even though the import path ends in `v2`


--

### Upgrading to a major version
#### Tidying it up

<br/>
`go.mod` will look like this

```
module mod
require github.com/robteix/testmod v1.0.1
require github.com/robteix/testmod/v2 v2.0.0
```
<br/>
<small>By default, Go does not remove a dependency from go.mod unless you ask it to</small>
```shell
$ go mod tidy
```
<small>Now we’re left with only the dependencies that are really being used</small>

---

### Using different versions

```go
package main
import (
    "fmt"
    "github.com/robteix/testmod"
    testmodML "github.com/robteix/testmod/v2"
)

func main() {
    fmt.Println(testmod.Hi("Roberto"))
    g, err := testmodML.Hi("Roberto", "pt")
    if err != nil {
        panic(err)
    }
    fmt.Println(g)
}
```

---

### Vendoring

- Go modules ignore the `vendor/` directory by default
 - The idea is to _eventually_ do away with vendoring

<br/>
```shell
$ go mod vendor
```
<small>
`go build` will ignore the contents of this directory by default
</small>
```shell 
$ go build -mod vendor
```
<small>
will build dependencies from the `vendor/` directory
</small>

--

### Vendoring
<small>
- Go module system is moving away from the idea of vendoring
 - Towards using a Go module proxy
- For those who don’t want to depend on the upstream version control services directly
- There are ways to guarantee that go will not reach the network at all (e.g. `GOPROXY=off`)

---

### Conclusion
<br/>
- Go module is basically transparent
- Import packages like always in our code and the `go` command will take care of the rest
- Dependencies automatically fetched when building
- No need for `$GOPATH` anymore
- Go module proxy may be the plan for the future

---

### Thank you!