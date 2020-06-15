---
title: "Build a Simple Deployer with Go by Using the Github Webhooks"
date: 2020-05-11T16:58:01+08:00
draft: false
tags:
    - go
categories: 
    - programming
---


In my journey to learn Go, I decided to start write about the topics that I'm learning. This technique known as Feynman Learning Technique. I have done this long time ago when I wanted to learn PHP, I started a blog in Persian, and start to write about everything I was learning about PHP. These articles might not be useful for experienced programmers, but it helps me to have a deep understanding about the stuff I'm going to learn.

In this article, I want to write a simple deployer by using Github Webhooks. 

Anyway, please let me know if you find any issues or mistakes. We can disscuss about it and we can learn together.


<!--more-->

At the end of this article, we will learn:

- How to setup Github Webhooks
- How to Parse and authenticate webhooks requests
- How to parse YAML and JSON
- Run a series of command in a specific directory


### Prerequisites

- Basic knowledge of Go
- Go must be installed in your machine
- GitHub account

## Table of content

## What do we expect from this project?

It's good to know what we should expect at the end of this project. Here is a short description.
We have a project in GitHub and we need a project to automate our deployment proccess whenever someone **push** something to the master branch. For this purpose, we are going to take advantage of GitHub webhooks which sends a request to a URL (apparantly in our sever) with a bunch of data about the changes that just made into the repository. The webhook can trigger after `push`, `merge`, `pull requests` and etc. For the sake of simplicity we just use `push` event for this tutorial. 

[Read more about Webhooks](https://developer.github.com/webhooks/)


## Configure GitHub webhooks

For using Webhooks, first, we need to enable and configure it for our repository. Open your github repository page and chose the Webhooks from the left sidebar. After click on Add webhook you will see the following page:


- Payload URL: The URL we want to send the request
- Secret: A secret key to authorize incoming the request
- Which events would you like to trigger this webhook?: In this section you can chose `push` or another events. For this tutorial we will only go with the `push` event.

And at the end, by selecting `Active` and `Add Webhook` this webhook will be enabled for our repository. Which means, everytime we `push` to the repository, GitHub will send a request to Payload URL. 



## Create a project and install needed packages

Create a new directory and call it whatever you want:

```sh
$ mkdir deployer && cd deployer
```

The only library we need is Yaml parser because we want to store some configuration in it.

```bash
go get github.com/go-yaml/yaml
```

## Deployer Configuration

For the sake of flixibility I decided to have a configuration in Yaml format. By this method, we can have multiple projects with different configurations. Create a `config.yaml` in your root directory:

```yaml
projects:
    - name: "username/project" 
      dir: "/path/to/blog"
      commands:
        - git status
        - git pull origin master
        - git submodule sync
        - git submodule update
      secret: the_secret_key
```

Our `config.yaml` file is consists of a `projects` array, and each project can have 
 - `name` which must be the same as the name of your github username and project
 - `dir` is the path of your project 
 - `commands` is a list of commands we need to run whenever a webhook reached our server
 - `secret` is the secret key you chose in your github.


### Parse config.yaml
Next step we are going to parse this file with helps of go-yaml/yaml package.

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


// Return a Project based on its name
func (c *Config) getProject(name string) *Project {
	for _, p := range c.Projects {
		if name == p.Name {
			return p
		}
	}
	return &Project{}
}

```

Thers is nothing much happened here, first we read the file and `Unmarshal` the content. The `getProject` function receives a name and return a `Project` instance.

## Parse incoming request

As soone as someone we `push` something to the repository, GitHub sends a request to the URL we set for webhook. The request body is contains a `payload` in `json` which we are going to parse the it here. Notice that, we only parse the information in that we need for this article, you can customize it based on your needs.

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
	// Repository name
	Repo    Repository `json:"repository"`

	// Slice of commits
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

To handle webhooks, we need a route and we are going to use built in go `http` library. 

Create `main.go` file with the following content:

```go

package main

import (
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/deploy", deployHandler)
	log.Fatalln(http.ListenAndServe(":3000", nil))
}

func deployHandler(w http.ResponseWriter, r *http.Request) {
	// 1. parse the payload
	// 2. parse config
	// 3. authenticate
	// 4. run commands
}
```
As you can see inside the `deployHandler` I mentioned 4 steps. Let's complete the steps on by one.



```go
	// Read the request body
	body, err := ioutil.ReadAll(r.Body)
	if err != nil {
		panic(err)
	}

	// 1. parse the payload
	payload, err := NewPayload(body)
	if err != nil {
		panic(err)
	}

	// 2. parse config
	config, err := NewConfig("./config.yaml")
	if err != nil {
		panic(err)
	}

	// get the project config based on the repository name
	project := config.getProject(payload.Repo.Name)

```

Most of the codes here are self explainatory, but notice how we read the request body at the first of the function.

### Authentication

In the next step we are going to authenticate the request. For this matter, we need to 

1. Read the `X-Hub-Signature` header which is a `sha1` encoded hash
2. Encode the request body with the secret key
3. Compare number 1 and 2

in `main.go` add a function and call it `isValidSignature`:

```go
func isValidSignature(body []byte, signature string, key string) bool {
	gotHash := strings.SplitN(signature, "=", 2)
	if gotHash[0] != "sha1" {
		return false
	}

	hash := hmac.New(sha1.New, []byte(key))
	if _, err := hash.Write(body); err != nil {
		log.Printf("Cannot compute the HMAC for request: %s\n", err)
		return false
	}

	expectedHash := hex.EncodeToString(hash.Sum(nil))
	return gotHash[1] == expectedHash
}
```

`isValidSignature` recieves three arguments, the request body in `[]byte`, the `X-Hub-Signature` header value, and the secret key. `isValidSignature` first checks the the hash algorithm which must be `sha1`, then encode the request body with the secret key, and at the end it compare the signature with the hashed value.

Let's add step 3 to our `deployHandler`:

```go
	....
	// get the project config based on repo name
	project := config.getProject(payload.Repo.Name)

	// 3. authenticate
	if !isValidSignature(body, r.Header.Get("X-Hub-Signature"), project.secret) {
		w.WriteHeader(http.StatusBadRequest)
		w.Write([]byte("error"))
	} else {
		// run the commands
	}
```


### Runner

The main purpose of this project is to run a sequence of commands whenever a github repository get updated. We already stored these commands in `config.yaml` file, now, it's time to run commands one by one and log the result.

Create a `runner.go` file and put the following code on it:

```go

package main

import (
	"log"
)

type Runner struct {
	// Commands stores a list of commands 
	Commands []string

	// Dir is the path Commands must execute
	Dir      string

	// Log is log the output of each command
	Log      *log.Logger

	// ErrLog is log all the errors
	ErrLog   *log.Logger
}

func NewRunner(commands []string, dir string) *Runner {
	return &Runner{
		Commands: commands,
		Dir:      dir,
		Log:      log.New(os.Stdout, "Log: ", log.LstdFlags),
		ErrLog:   log.New(os.Stdout, "Error: ", log.LstdFlags),
	}
}

```
`Runner` struct is reponsible to run a sequence of commands in an specific directory. By default it logs data to Stdout, but we can write the logs into a file with something like this:

```go

r := NewRunner(project.commands, project.Dir)

f, _ := os.OpenFile("runner.log", os.O_RDWR | os.O_CREATE | os.O_APPEND, 0666)
r.Log.SetOutput(f)

fe, _ := os.OpenFile("error.log", os.O_RDWR | os.O_CREATE | os.O_APPEND, 0666)
r.ErrLog.SetOutput(fe)

```

Now add the following functions in `runner.go`:

```go

func (r *Runner) run() error {
	for _, command := range r.Commands {
		if err := r.exec(command, r.Dir); err != nil {
			return err
		}
	}
	return nil
}

func (r *Runner) exec(command string, dir string) error {
	c, args := r.parse(command)
	cmd := exec.Command(c, args...)
	cmd.Dir = dir
	output, err := cmd.Output()
	if err != nil {
		r.ErrLog.Println(c, err)
		return err
	}
	r.Log.Println(string(output))
	return nil
}


func (r *Runner) parse(cmd string) (string, []string) {
	c := strings.Split(cmd, " ")
	return c[0], c[1:]
}
```

`os/exec` package is responsible to run a system command. The first argument for `Command` function is the command we want to run, and the second parameter is a slice of string which is the command's parameters.

Based on our `config.yaml`'s `commands` section, the commands has written in one line that we need to take the first part as an actual command and the rest of it must consider as the command's parameters. We have done this in `parse` function. `exec` function is responsible for executing a command in the given directory, and finally, the `run` command will run all the commands we have sent to the `Runner` constructor.

Now it's time to finish our `deployHandler` function. Open `main.go` and add the last step to the `deployHandler` function:


```go
func deployHandler(w http.ResponseWriter, r *http.Request) {

	// 1. parse the payload
    ...

	// 2. parse config
	... 

	// 3. authenticate
	if !isValidSignature(body, r.Header.Get("X-Hub-Signature"), project.secret) {
		w.WriteHeader(http.StatusBadRequest)
		w.Write([]byte("error"))
	} else {
		// 4. run commands
		runner := NewRunner(project.Commands, project.Dir)
		if err := runner.run(); err != nil {
			w.WriteHeader(http.StatusBadRequest)
			w.Write([]byte("error"))
		} else {
			w.WriteHeader(http.StatusOK)
			w.Write([]byte("success"))
		}
	}
```

`runner.run()` runs commands one by one and return error in each step the running command failed.

Our deployer is working now. But there are many things can be added to it. 