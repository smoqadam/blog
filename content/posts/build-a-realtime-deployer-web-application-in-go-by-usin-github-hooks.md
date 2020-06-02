---
title: "Build a Realtime Deployer Web Application in Go by Usin Github Hooks"
date: 2020-04-11T16:58:01+08:00
draft: true
tags:
    - go
categories: 
    - programming
---

I'm sure you're agree with me that real-time web apps are so intersting and somehow challenging. In my journey to learn Go, this time, I decided to create a real-time web application. This project would be a simple deployer which works with Github webhooks. 

<!--more-->

At the end of this article, we will learn:

- How to create a web app by mux/router
- How to use gorilla/websocket to send/receive data to/from client
- How to use HTML5 websocket API to send/receive data to/from server
- How to parse YAML and JSON
- How to use Github webhooks


I'm already get excited, let's start.


### Prerequisites

- Basic knowledge of Go
- Go must be installed in your machine
- Basic knowledge of JavaScript
- GitHub account

## Table of content

## What do we expect from this project?

It's good to know what we should expect at the end of this project. Here is a short description.
We have a project in GitHub and we need a project to automate our deployment proccess whenever someone **push** something to the master branch. It should send a request to a URL (apparantly in our sever) and this project is going to run a several commands on the server. 

For this purpose, we are going to take advantage of GitHub webhooks which sends a request to a URL with a bunch of data about the change that just made into the repository. The webhook can trigger after `push`, `merge`, `pull requests` and etc. For the sake of simplicity we just use `push` event for this tutorial. 

[Read more about Webhooks](https://developer.github.com/webhooks/)


## Configure GitHub webhooks

For using Webhooks, first, we need to enable and configure it for our repository. Open your github repository page and chose the Webhooks from the left sidebar. After click on Add webhook you will see the following page:


- Payload URL: The URL we want to send the request
- Secret: A secret key to authorize incoming request
- Which events would you like to trigger this webhook?: In this section you can chose `push` or another events. For this tutorial we will go with only `push` event.

And at the end, by selecting `Active` and `Add Webhook` this webhook will be enabled for our repository. Which means, everytime we `push` to the repository, GitHub will send a request to Payload URL. 



## Create a project and install needed packages

Create a new directory called whatever you want:

```sh
$ mkdir deployer && cd deployer
```

The only library we need is Yaml parser because we want to store some configuration in it.

```bash
go get github.com/go-yaml/yaml
```

## Deployer Configuration

It's better to have a config file to have more control over our application. To acheive this, we are going to store configuration in Yaml file. Create a `config.yaml` in your root directory:

```yaml
projects:
    - name: "smoqadam/blog" 
      dir: "/home/saeed/projects/blog"
      commands:
        - git status
        - git pull origin master
        - git submodule sync
        - git submodule update
        - hugo
      onFailure: ./notif.sh
```

Here is our configuration. It has a projects array, `name` of the repository, the path and the commands you want to run, and `onFailure` which runs whenever something goes wrong in the `commands` section.

That's all, let's parse this file with the help of the `go-yaml` package. Create a file and call it `config.go` with the following contents:

```go

package main

import (
	"io/ioutil"

	"gopkg.in/yaml.v2"
)

type Project struct {
	Name      string   `yaml:"name"`
	Dir       string   `yaml:"dir"`
	Commands  []string `yaml:"commands"`
	OnFailure string   `yaml:"onFailure"`
}

type Config struct {
	Projects []*Project `yaml:"projects,omitempty"`
}

func NewConfig(path string) (*Config, error) {
	c := &Config{}
	y, err := ioutil.ReadFile(path)
	if err != nil {
		return c, err
	}
	yaml.Unmarshal(y, &c)
	return c, nil
}

```

First we read the file then parse it with `yaml.Unmarshal` and return the struct.


## Parse incoming request

Whenever an event triggers, github will send a request to Payload URL with a bunch of information about the repository in Json format. 

Create a file and call it `payload.go`:

```go
package main

import (
	"encoding/json"
)

type Author struct {
	Name     string `json:"name"`
	Email    string `json:"email"`
	Username string `json:"username"`
}

type Commit struct {
    ID       string   `json:"id"`

    // commit message
	Message  string   `json:"message"`
    
    // commit author
    Authod   Author   `json:"author"`
    
    // list of the files that added to the repo
    Added    []string `json:"added"`
    
    // list of the files removed from the repo
    Removed  []string `json:"removed"`
    
    // list of the files modified in the repo
	Modified []string `json:"modified"`
}

type Repository struct {
    // repository name e.g. username/repo 
	Name string `json:"full_name"`
}

type Payload struct {
	Repo    Repository `json:"repository"`
	Commits []Commit   `json:"commits"`
}

func NewPayload(input []byte) (*Payload, error) {
	p := &Payload{}
	if err := json.Unmarshal(input, &p); err != nil {
		return p, err
	}
	return p, nil
}

```

`Payload` struct has two fields, `Repo` that store information about the repository and the `Commits` that store array of `Commit` struct. 



## Create route

To handle webhooks, we need a route and we are going to use built in `http` library. 

Create `main.go` file with the following content:

```go

package main

import (
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
	"os"
	"time"
)

func main() {

	http.HandleFunc("/deploy", deployHandler)

	log.Fatalln(http.ListenAndServe(":3000", nil))
}

func deployHandler(w http.ResponseWriter, r *http.Request) {

    // parse config file as soon as the request received
	config, err := NewConfig("./config.yaml")
	if err != nil {
		panic(err)
	}

    // read the request body and parse the payload
	p, err := ioutil.ReadAll(r.Body)
	if err != nil {
		log.Fatalln(err)
	}
	payload, err := NewPayload(p)
	if err != nil {
        log.Fatalln(err)
	}
    
    // // get the project config
    // project := config.getProject(payload.Repo.Name)

	// deployer, _ := NewDeployer(project.Commands)
	// if err := deployer.run(); err != nil {
	// 	logger("Failed", []byte(err.Error()), "./error.log")
	// 	deployer.exec(project.OnFailure, project.Dir)
	// }
}

```

`deployHandler` is the function that runs everytime any request comes to `/deploy` route. Inside the function, first we read the `config.yaml` file because later, we can change the config without to restart the application.

`ioutil.ReadAll` reads the request body and returns a `[]byte` which we passed it the `NewPayload` function.   


## Runner

to handle and execute all commands inside `commands` array in `config.yaml`, we will create a struct called Runner. This struct receives a slice of commands and runs them one by one. 

Anyway, create a file and call it `runner.go`:

