## Why Go?

My programing language of choice is Ruby. I just love to work with it. As a former Java developer, Ruby has made my life so much better. Apart from the beautiful language, Ruby also has a great community. This more than anything is what makes Ruby so powerful.

So why would I be interested in a different language? Well much of it has to do with the Ruby community. Rubyists are often playing with other languages, especially new and exciting ones. And the buzz around Go has been increasing over the past 12 months in the tech community but also among  a good few Rubyists that I respect. And the buzz seems to be a lot more than a casual curiousness. So I decided to take a look. 

## Taking the tour

I have often found that the homepage of different technologies neglects the newbie. And that the best place to start is with a tutorial or introduction by some blogger that ranks high in a google search. This seems to be changing for newer technologies and thankfully Go is no different. The first thing you notice on the [golang.org](golang.org) site is the 'Try Go' section where you can actually type Go code and run it.

```go
import "fmt"

func main() {
	fmt.Println("Hello, 世界")
}
```

This 'Try Go' section is linked to the '[Tour](tour.golang.org)'. So I started there. The tour is 70 or so steps that guide you along from a simple 'Hello world' program to concurrent goroutines sending data over channels.

It is a really great way to get into Go and it will get you up to speed in no time. You quickly learn the syntax, basics of the language, and features that make Go stand out.

## Taking off the training wheels

With the introduction over I wanted to build something on my own. I have wanted a command line program that would spit out a list of open pull requests from Github for our organization. Along with some information about each pull request. So I decided to build it with Go.

## Env setup

Before I could start in on this new program I need to set up my environment.

### Installation

Installation is via a binary [download](https://code.google.com/p/go/downloads/list). 

You also need to set up some environment variables [see the documentation](http://golang.org/doc/install).

```
export GOROOT=$HOME/gocode
export PATH=$PATH:$GOROOT/bin
export GOPATH=/Users/wmernagh/gocode
```

## Where to work (play)

So my first mistake was to create a folder for my program in a non standard place. Go expects you to write your code in
```
$GOPATH/src/github.com/<username>/<program_repo_name>
```
if you are storing you code on github. (This is not entirely true but works well if you want to standardise.) 

### Building your app or package
Now you can build your program and install it as follows

```
$ go build <username>/<program_repo_name>
$ go install <username>/<program_repo_name>
```

If you are in the source folder you can omit the path

```
$ go build
$ go install
```

## Installing Third Party Packages

One of the nicest things about Go is the community, similar to Ruby, it has a great many packages that are open sourced. Also similar to Ruby, Go comes with a very simple way to get these packages. It works a lot like RubyGems.

When you run `go get X` it installs X in your `GOPATH`. It installs the packages in `pkg` and the source in `src`. When you use `go install X` it will put the binary in bin. This is true whether X is local or on a  remote server like Github or Google.

```
$ go get code.google.com/p/go.example/hello
```

That will download the hello package into $GOPATH and it can be imported into any program you now write.

```
$ tree $GOPATH -L 2
/Users/wmernagh/gocode
├── bin
│   ├── gogit
│   └── hello
├── pkg
│   └── darwin_amd64
│       ├── code.google.com
│       │   └── p
│       │       ├── go.example
│       │       │   └── newmath.a
└── src
    ├── code.google.com
    │   └── p
    │       ├── go.example
    │       │   ├── LICENSE
    │       │   ├── PATENTS
    │       │   ├── codereview.cfg
    │       │   ├── hello
    │       │   │   └── hello.go
    │       │   └── newmath
    │       │       ├── sqrt.go
    │       │       └── sqrt_test.go
```

Then you can include a package as follows

```go
package main

import "code.google.com/p/go.example/hello"

func main() {
...
```

[How to Write Go Code](http://golang.org/doc/code.html) is great on introducing you to the different aspects of project layout and package dependancies.

#### Hg

Depending on the packages you try to install for Go you will likely need to install Mercurial (I am assuming you already have Git installed).

This can be done with brew on a Mac or you can just download it from the [mercurical](http://mercurial.selenic.com) site.

### My First Program

As I mentioned earlier my program will fetch all open pulls for an organization or user on Github.

```plain
$ gogit -owner IoraHealth

snowflake (2)
| Pull | Comments | Passing | :octocatted: |
|  142 |        1 | failure |         true |
|  152 |        2 | failure |         true |

salk (4)
| Pull | Comments | Passing | :octocatted: |
| 2085 |        0 |         |        false |
| 2011 |        3 |         |         true |
| 2089 |        1 |         |         true |
| 2087 |        0 |         |        false |

bouncah (3)
| Pull | Comments | Passing | :octocatted: |
|  157 |        1 |         |         true |
|  158 |        1 |         |         true |
|  127 |        1 |         |         true |
```

This was a perfect problem for which to learn some Go. I would have to hit a number of Github API endpoints so I could also take advantage of Go's straightforward concurrency support.

To print out the details the program needs to get:

 - a list of repositories for the provided Organization
 - a list of open pull requests for each repo
 - a list of comments for each open pull
 - the pulls status for each open pull

That is a lot of work and could be slow if not done concurrently.

#### Basic structure

I anticipate reusing some of this code in other programs. So I decided to split the program into two packages. A __gogit__ package and a __main__ package.

In Go, __main__ packages are what get built into executables. All other packages are libraries to be included in __main__ or other packages.

```go
package main

import (
	"github.com/wm/gogit"
	"fmt"
	"syscall"
	"flag"
)

var opts map[string]string
var ok bool

func main() {
	opts, ok = readOptions()

	if !ok {
		printUsage()
		return
	}

	gogit.SetGithubToken(opts["TOKEN"])
	repos, _ := gogit.ListRepos(opts["OWNER"])

	c := make(chan []gogit.Pull, len(repos))

	for _, repo := range repos {
	    if repo.OpenIssuesCount > 0 {
			go getReposOpenPulls(repo, c)
		}
	}

	for _, repo := range repos {
	    if repo.OpenIssuesCount > 0 {
			printRepoPulls(c)
		}
	}
}
```

The `main` package uses the `gogit` package and does only a few things:

1. Reads the Github API token to use
2. Reads the Organization or User to scope to
3. Gets a list of repositories for that scope
4. Gets the open pull requests for each repository
5. Display information for each set of open pull requests

So 1 and 2 are straightforward. In 3 it sets the gogit Github token and gets the repositories from Github using my gogit library.

### Concurrency

But 4 and 5 are where the first bits of magic happen.

After fetching the repositories it iterates over them placing slow method calls into their own `goroutines`.

```go
	for _, repo := range repos {
	    if repo.OpenIssuesCount > 0 {
			go getReposOpenPulls(repo, c)
		}
	}
```

```go
func getReposOpenPulls(repo gogit.Repo, c chan []gogit.Pull)  {
	pulls, _ := repo.OpenPulls()
	c <- pulls
}
```

It runs `getReposOpenPulls` within a `goroutine` . `getReposOpenPulls` gets the `OpenPulls` for the provided Repo and sends them to the provided channel (more on channels in a bit). Since `OpenPulls` needs to hit Github's API (a number of times) it will be slow. By running it in a `goroutine` we can call `getReposOpenPulls` on multiple repos concurrently. Each time we call `go` the function passed will be executed in its own `goroutine`. It will not start running until the current `goroutine` is blocked.

The main execution path is considered a `goroutine`. So you are always running at least one `goroutine`.

The first time our program blocks is within the first call to `printRepoPulls` here:

```go
	for _, repo := range repos {
	    if repo.OpenIssuesCount > 0 {
			printRepoPulls(c)
		}
	}
```

```go
func printRepoPulls(c chan []gogit.Pull) {
	pulls := <-c
	pullCount := len(pulls)
...
```

It waits for an array of `Pulls` to be sent along the channel so it blocks until that happens.

This gives our first call to `getResposOpenPulls` a chance to run, and it does until it blocks. It likely blocks on IO when hitting the GitHub API, giving the next non-blocking `goroutine` a chance and so on.

Then once one of the `getReposOpenPulls` `goroutines` sends data to the channel `printRepoPulls` continues and the loop hits `printRepoPulls` again and so on.

### Channels

Channels are used to pass data between `goroutines	`. The channel needs to have a type associated with it. And you need to allocate memory with make. 

```go
	c := make(chan []gogit.Pull, len(repos))
```

Here I create a buffered channel the same length as the number of repos. You can also create a synchronous channel, which is the default if you do not specify the length or you specify a length of zero.

From [Effective Go](http://golang.org/doc/effective_go.html#channels):
> Receivers always block until there is data to receive. If the channel is unbuffered, the sender blocks until the receiver has received the value. If the channel has a buffer, the sender blocks only until the value has been copied to the buffer; if the buffer is full, this means waiting until some receiver has retrieved a value.

#### Digging deeper

I also use `goroutines` within the `OpenPulls` method since it needs to fetch different data for each pull.

```go
func (repo *Repo) OpenPulls() (result []Pull, err error){
	ghPulls, _, err := client.PullRequests.List(
		repo.Organization, repo.Name, nil)
	if err != nil { return nil, err }

	return updatePulls(repo, &ghPulls), nil
}

func updatePulls(repo *Repo, ghPulls *[]github.PullRequest)([]Pull) {
	pulls := make([]Pull, len(*ghPulls))
	c := make(chan Pull, len(pulls))

	for _, ghPull := range *ghPulls {
		go updatePull(repo, ghPull, c)
	}

	for i := 0; i < cap(c); i++ {
		pulls[i] = <-c
	}

	return pulls
}

func updatePull(repo *Repo, ghPull github.PullRequest, c chan Pull) {
	pull := Pull{&ghPull, nil, repo}
	pull.Update()

	c <- pull
}
```

You can take a look at more all the code at [github.com/wm/gogit](https://github.com/wm/gogit/tree/wm-blog_post)

[Warning]: Excuse my sloppy code - I am just hacking. And this is still very much a WIP!

## Conclusion

I am really excited by Go. Even though it is a typed language it is very approachable and you quickly get comfortable with it.  You feel like you are writing in a dynamic language like Ruby because of the implicit typing that Go does for you. 

The other nice thing is the ease in which you can build concurrent programs. It is pretty amazing. I know you can include libraries in Ruby but Go has concurrent primitives. And their use is straightforward.

The documentation is great and there is plenty of best practice advice out there. Effective Go is a great start for this. 

The community too looks promising. There are already many great projects on Github. [Martini](http://martini.codegangsta.io) is a framework a lot like Ruby's Sinatra. I for one will be writing something in with that soon.

That said some things irk me with Go. I am still not loving the package system. I think that is in part because I am unsure exactly how to organize my source code. But this uncertainty is a downer for me. I had similar issues with Ruby when I was starting out. So I probably just need to see a pattern in the community that I like.

I will be using Go for all my side projects in the future in an effort to get better at it.

## Resources

The [golang.org](http://golang.org) site has many great resources. Getting started with Go I found the following really useful:

- [The Tour](http://tour.golang.org)
- [Effective Go](http://golang.org/doc/effective_go.html)
- [The minimum you need to know about arrays and slices in Go](http://openmymind.net/The-Minimum-You-Need-To-Know-About-Arrays-And-Slices-In-Go/)
- [Go for Rubyists](http://www.sitepoint.com/go-rubyists/)


Some parting words from Rob Pike one of the creators of Go

> Go aims to combine the safety and performance of a statically typed compiled language with the expressiveness and convenience of a dynamically typed interpreted language.
