---
 title: "Simple auto deployer with PHP"
 date: 2018-12-15T16:16:50Z
 draft: false 
---

How you deploy your projects? Are you using fancy things like Jenkins with lots of features and configuration that you don't even know about? For one of my personal projects, I needed to have a deployment tools. I tried to install and configure Jenkins but there was no chance to make it up and running. So I decided to write my own, simple deployer with PHP. 

### Git Hooks

For this article we are using Git hooks. Git hooks is a script that Git runs before or after some events such as: commit, push, merge, create a branch and so on. Although, web-based Git managers such as Github, Gitlab or bitbucket call it Webhooks, it means that the events call a URL (usually in your server) instead of running a script. 


### configure webhooks 

first of all, we need to config our repository to use webhooks. For this purpose, go to your repository settings and you will find `Webhooks` on the sidebar.

![](/github-settings.png)

After clicking on `Add webhook` button you will see the webhook settings page: 

![](/webhook-settings.png)

__Payload URL__: the URL you want to send a POST request after or before an event

__Content Type__: specify how you want to receive data from GitHub

__Secret__: a secret key used for authorization

__Which events would you like to trigger this webhook?__: 

* __Just the push event__: This option just send event name to the callback URL
* __Send me everything__: send everything to callback URL included repository and user information
* __Let me select individual events__: By choosing this option you are able to specify which events  do you want to follow


For this article, we stick to the first option. You can play with other options when you get familiar with the whole process.


### Deployer

In the previouse step we set the push event for our webhook. It means as soon as we push something to the repository, the following http request send to the payload url:

```
Request URL: http://mydomain.com/deployer.php
Request method: POST
content-type: application/x-www-form-urlencoded
Expect: 
User-Agent: GitHub-Hookshot/233462e
X-GitHub-Delivery: ff42fb42-0107-11e9-961e-9dc5ae0da8b1
X-GitHub-Event: push
X-Hub-Signature: sha1=81407666112671798a83c608efa69d1052987fe9
```

as you can see, there are three custom headers which set by Github. For authorization, we must use 
`X-Hub-Signature` header. value of `X-Hub-Signature` header is consists of a hash algorithm and a hash string.

Create a file called `deployer.php` and put the following content on it:


```php
<?php

$secret = 'MYSECRETKEY';

if (!isset($_SERVER['HTTP_X_HUB_SIGNATURE'])) {
    exit("Permision denied!");
}

```
Variable **$secret** is the secret key you had choosen in webhook settings. We need to compare value of `HTTP_X_HUB_SIGNATURE` header with `sha1` value of POST request content. For this we used the following code:


```php
$githubHeader = $_SERVER['HTTP_X_HUB_SIGNATURE']; 
$rawPost = file_get_contents("php://input");
$secret = str_replace("sha1=", "", $githubHeader);
$hash = hash_hmac('sha1', $rawPost, $mySecret);
if ($hash != $secret){
    exit ("Permission denied!");
}
```

First, we remove `sha1=` string from the header value then convert the post content to hash with `sha1` algorithm after that we compare two values.  


```php
$workTree = 'PROJECT_PATH';
$gitRoot = $workTree. '/.git';

$commands = array(
    'pull',
    'status',
    'submodule sync',
    'submodule update',
    'submodule status',
);

$log = [];
$log[] = "####### ". date('Y-m-d H:i:s') ." #######";
foreach($commands as $cmd) {
    $cmd = 'git --git-dir '.$gitRoot.' --work-tree '. $workDir . ' '. $cmd
    $tmp = shell_exec("$cmd 2>&1");
    $log[] = "\$ $cmd\n".trim($tmp)."\n";
}
$log = implode("\n", $log);
file_put_contents ('deployer.log', $log, FILE_APPEND);
exit('success');
```

__$workTree__ value must point to our project directory. As soon as we push new changes to Git repository, GitHub sends a POST request to payload URL and above script will be run. After authentication, __git pull__ and __git status__ commands run and the output will be stored in _deployer.log_ file.
