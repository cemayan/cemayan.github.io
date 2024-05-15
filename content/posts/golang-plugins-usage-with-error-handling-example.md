---
title: "Golang plugins usage (with error handling example)"
date: 2024-06-22T00:00:00+00:00
weight: 1
# aliases: ["/first"]
tags: ["golang","plugins","error-handling"]
author: "cemayan"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
disableShare: false
disableHLJS: false
hideSummary: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
editPost:
    URL: "https://github.com/cemayan/cemayan.github.io/tree/main/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---



## Fundamentals

Package plugin implements loading and symbol resolution of Go plugins.

**Pros**:
- _**Dynamic Loading**_: Plugins can be loaded and unloaded dynamically at runtime.
- _**Encapsulation**_: Since plugins run under their own package, there are no conflicts etc.

**Cons**:
- _**Version Dependency**_: While I was using it, I kept getting the error 'plugin was built with a different version of package XXX' because the plugin.so file requires all packages used in the build to be the same as the main application. It can be a bit annoying, but after a bit of thought I can see that it's almost necessary (better to get an error now than in runtime :))

You can found the further warnings [here](https://pkg.go.dev/plugin#hdr-Warnings) 

---

## Introduction

In order to develop a plugin for main app you need to create a go file like below:




**group.go**:
```go
package errorg

var symName = "Group"

// Group represents interface that will be implemented by plugins
type Group interface {
   GetErrorMap() map[string]interface{}
}
```



**plugins/plugin.go**:
 ```go 
package  main   

type httpError struct {
}

func (g httpError) GetErrorMap() map[string]interface{} {
	return map[string]interface{}{
		"400": struct {
			Message string `json:"msg"`
		}{"The server cannot or will not process the request due to something that is perceived to be a client error (e.g., malformed request syntax, invalid request message framing, or deceptive request routing)."},
	}
}

var Group httpError
```
---

📝️ _**Useful Notes**_:
- package must be main.
- method name must be same with that will be implemented main app's interface method name.
- if return type is not strict such as interface, you can convert the value in main app.
- golang version and libraries version must be same.


Now we can build to our plugin:


```shell
go build  -buildmode=plugin   -o  plugins/plugin.so plugins/plugin.go
```

This command creates a **plugin.so** file. (In Linux, a .so file is a Shared Object file, also known as a shared library)


🎉 Finally, we can add the plugin to main app.


**group.go**:
```go
package errorg

import (
    "fmt"
    "plugin"
    "errors"
)

var symName = "Group"

// Group represents interface that will be implemented by plugins
type Group interface {
  GetErrorMap() map[string]interface{}
}

// loadThePlugin loads the plugin according to the given path
// pluginPath is "plugins/plugin.so"
func loadThePlugin(pluginPath string) error {

    // Open opens a Go plugin.
    plug, err := plugin.Open(pluginPath)
    if err != nil {
     return errors.New(fmt.Sprintf("an error occurred while opening plugin: %v", err))
    }
	
    //Lookup searches for a symbol named symName in plugin p.
    sym, err := plug.Lookup(symName)
    if err != nil {
     return errors.New(fmt.Sprintf("an error occurred while looking plugin: %v", err))
    }

    var group Group
    group, ok := sym.(Group)
    if !ok {
     return errors.New("unexpected type from module symbol")
    }

    errorMap := group.GetErrorMap()
    // do some stuff
    fmt.Println(errorMap)
	
    return nil
}
```

Thus, our code is loaded as plugin.

---

## Real life usage in app

errorg helps to error handling for golang apps and  thanks to golang/plugin it can be added the new plugin
The main purpose of this library is to manage all error codes from a common library.


Clone the repo:

```shell    
git clone https://github.com/cemayan/errorg.git
```

Example main.go:
```go
package main

import (
    "fmt"
    "github.com/cemayan/errorg"
)

func test() error {
    return errorg.New("http", "401")
}

func init() {
    errorg.Init(errorg.JsonLog())
}

func main() {
    fmt.Println(test())
}

```


In order to see result on example app make commands can be used like below:

```shell
make init
make example
```

And example app returns like below:
```json
{"msg":"Although the HTTP standard specifies \"unauthorized\", semantically this response means \"unauthenticated\". That is, the client must authenticate itself to get the requested response."}
```

---


## Conclusion

It is not widely used, but I think it can be used in some cases because it adds a flexible structure* to the app.


- Plugin can be loaded/unloaded easily at runtime
- You can add new plugin whenever what you

📚 Further readings:
 

**_Real life project_**:

- https://tyk.io/docs/plugins/supported-languages/golang


**_RPC based plugins_**:

- https://github.com/hashicorp/go-plugin
- https://eli.thegreenplace.net/2023/rpc-based-plugins-in-go/

**_Optimization_**:

- https://sonyarouje.com/2021/04/25/go-plugin-file-size-optimization/

Thank you for reading. I'm waiting for your questions and comments.