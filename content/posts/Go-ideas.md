Have you ever been out of idea on what to code for your next open-source or side-project? As a beginner, you can find many resources and tutorials on different beginner-friendly projects out there. But when you pass that level, you just cannot come up with any interesting ideas to work on while you’re learning.

If you’re a type of person who likes to learn by doing, here I’m gonna list some of my ideas that wrote down and I was supposed to work on while I was learning Go. 
Some of them has started, but haven’t finished. You can take any of these ideas for you next project or if you think these are so easy for you, try to write an article about it.

> Note:The purpose of this list is to make something useful while you’re learning and It doesn’t matter if some of them already exists.


## Download manager with web and CLI interface

Features:
Download a file with multiple connections
Be able to resume
Show downloading progress on CLI and Web 

Try to break things down for yourself. For instance, start by learning how to read a file on the internet. Then, try to write the file on the disk. By doing these two steps, officially you’d have a downloader. Now try to improve it by adding and improving it step by step.

### You’d learn:
- Work with HTTP related packages/libraries to read file from a remote address
- Work with file systems to write file on the disk and have resume ability
- Goroutine to manage multiple connections and writing on the dist at the same time
- Work with CLI I/O to parse input arguments and show a proper progress bar
- Work with web-related functions/libraries to create a web interface 


## Implement ZIP algorithm in GO
This can be quite interesting and boring at the same time. Maybe you say what’s the point of implementing it in the first place. I’d say to learn. Doesn’t matter what we code, the goal here is learning.
 
### You’d learn 
- how ZIP algorithm works
- work with file systems
- how to read RFC’s (in this case [ZIP RFC](https://www.ietf.org/rfc/rfc1951.txt))




## Pretty panic()

First time when I saw the panic() message on my terminal I’d been trying to figure out what is it saying. Still, it takes me a while to realize what is the error pointing out. So by creating a library, you would save many people like me. The idea is to show the message for panic() in a beautiful and more meaningful way.

#### You’d learn:
- Error handling
- String manipulation


## Alarm clock, reminder web/CLI
A CLI or web app to add alarm clock or reminder with a calendar. If you think it’s easy for you, try to write an article about it.

#### You’d learn:
- how to work with time package
- how to manage terminal’s input and output
- how to deal with web related stuff (http, templates)



## Go database manager (something like Adminer)

[Adminer](https://www.adminer.org/) is my favourite database manager amongst others, but the idea that PHP needs to be install on the server to run Adminer could be overwhelming for some people. It’d be nice if I could upload a single binary to my server, and run it on a specific port to have a simple, nice, and fast web-based database manager. I started [this](https://github.com/smoqadam/dbman) project a while back, but I didn’t continue.

#### You’d learn:
- web related libraries and stuff
- SQL syntax
- managing a database for browsing, and managing a database, table, row or column
- how to manage security issues that might happen when you’re working with DB
- and many other stuff


## CLI file manager

I really like the idea behind [nnn](https://github.com/jarun/nnn), a file manager that runs inside the terminal. I [started](https://github.com/smoqadam/go-filemanager) to work on this one but didn’t continue. I’m sure you’d learn many things from this project.

#### You’d learn:
- work with file systems (list, copy, delete, paste, read or open)
- create a UI for your CLI application
- Goroutins to manage background task (e.g. copy a large file)



## Go deployer
A few month ago I wrote an article on [how to create a simple deployer by Go and GitHub webhooks](https://smoqadam.me/posts/build-a-simple-deployer-in-go-by-usin-github-hooks/) . 
You can check my article, but you can add more features like web UI, send notification if the job has failed.  

#### You'd learn:
- how to run bash commands inside your Go application
- how to parse and verify http request

