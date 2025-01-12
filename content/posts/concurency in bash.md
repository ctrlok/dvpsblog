+++
title = "Concurrency in Bash"
decription = "Learn how to achieve efficient concurrency in Bash with practical examples, including Docker image builds, parallel pushes, and webhook calls. This witty guide covers pipes, background processes, and even compares Bash with Go for handling complex tasks."
date = "2025-1-5"
[taxonomies]
tags = ["bash", "concurrency", "linux", "pipes"]
[extra]
comment = true
hidden = false
+++

In the days of yore, every interview I ran was a delightful circus of chaos: concurrency, Linux kernel deep dives, gdb/strace (and later eBPF and flamegraphs), and, of course, the classic “so, can you actually use the tools you listed?” quiz, which often includes Bash-related questions. 

Ah, simpler times. Anyway, I asked those questions for a reason:
 
* It never failed to irritate so-called “senior **YAML** architects.” 
* It separated the groovy cats from those who just memorized buzzwords.
* Admittedly, I was a far more _insufferable_ young man than the refined gentleman I am today.

Also, deep down, I think I tried to form a squad of superhero ninja turtles—only to see them slog through soul-crushing configurational drudgery. 

That experience truly helped me appreciate engineers who were just happy to configure Jenkins and install a Helm chart.
Funny how life teaches *humility*!

I’ve learned my lesson and no longer ask those questions. However, I still get a strange satisfaction from a meticulously crafted script—especially since reproducing the same logic in another language would be much more challenging.

And that, dear reader, is precisely what I want to talk about.

# The Bash
{{ images(position="right", path="/images/bash/pyr.jpg", width = 250, margin=15) }}

Look, I get it—really, I do. Bash is clunky, ancient even, designed when people were building pyramids (and some of its creators might already be buried in them). 
It’s not the sleekest language on the planet.

But here’s the thing: the people who created Bash were **brilliant**. 
They came up with **clever** and, more importantly — **practical** solutions.  

And I genuinely believe that many modern engineers could benefit from learning Bash, at least enough to avoid reinventing the wheel each time they need to do something more advanced than the `ls | grep file.`


# The Task

{% mermaid() %}
---
config:
  theme: base
  look: classic
  layout: dagre
  themeVariables:
    lineColor: '#fff'
    borderRadius: 5
    primaryBorderColor: 'rgb(31, 31, 31)'
    fontFamily: 'JetBrains Mono'
---

flowchart LR

list["List of Images"] --> build["docker build"]
build --> worker1["push worker 1"] & worker2["push worker 2"]
worker1 --> webhook["webhook"]
worker2 --> webhook
list@{ shape: procs}
style list fill:#dd72cb
style build fill:#5ec8df
style worker1 fill:#e3e3f9
style worker2 fill:#e3e3f9
style webhook fill:#49a1f5

{% end %}


It happened that I stepped into a pile of task (simplified version): 

* Build Docker image.
* Push this image in a few Docker registries.
* Call webhook right after. 

{% note(header="The real task I made (clickable)", clickable=true, hidden = true) %}

1. Build huge Docker images with AI models
2. Export them one by one to OCI format (using `skopeo`)
3. Convert them from OCI to remote-registry format  (using a self-written script)
4. Run `rclone` on the blobs to copy them to multiple registries around the world
5. Convert them back to OCI (so `skopeo` will be able to reuse cache later)
6. Push images to update manifests with already pushed blobs

**Limitations:**
* Only a single image can be built at the time
* Push operations are exclusive to every registry

{% end %}

The naive solution would be:

```bash
#!/bin/bash -e

images=("image1" "image2" "image3")

for image in "${images[@]}"; do
  docker build -t "registry-1/$image" -t "registry-2/$image" .
  docker push "registry-1/$image"
  docker push "registry-2/$image"
  curl -X POST "https://example.com/webhook/$image"
done
```

Simple. Elegant. Deadly. But not optimal:

* We can build `image2` while pushing `image1`
* We can push images in parallel

{% note(header="Warning") %}

Async ⊆ concurrency and almost always — *optimization*. And ‘Premature optimization is the root of **all evil**’ © Donald ~~Duck~~ Knuth.

Instead, naivety is often *sufficient*: the simpler the idea, the easier it is to read and maintain.

If the downsides of a naive approach aren’t critical, then always pick the naive path with minimal optimizations.

Remember: one day, your code might be read by a JavaScript developer… and they might not survive the experience.

{% end %}

# Concurrency

I met a lot of engineers who praise Go for its goroutines and channels, yet were unaware that you can run any process in the background using the `&` symbol:

```bash
# This task runs in the background  
some_task &  

# So, does this one  
(  
    other_task  
    sleep 200  
    some_webhook  
)&  

# You can wait for a specific task using its PID  
last_task_pid=$!  
wait $last_task_pid  

# Or wait for all background tasks to finish  
wait  
```

The simplest concurrent version of our script would look like this: 

```bash
docker_build &
docker_push reg1/$image &
docker_push reg2/$image &
webhook $image &
wait
```

{% note(header="Why to create functions?", clickable=true, hidden = true) %}
Functions are much easier to debug and extend. 

```bash
function docker_build() {
    echo Start build for $1
    sleep 0.3
    echo End build for $1
}

function docker_push() {
    echo Start push for $1
    sleep $2
    echo End push for $1
}

function webhook() {
    echo Start webhook for $1
    sleep 0.1
    echo End webhook for $1
}

# Don't forget to export them so they can be used in forks
export -f docker_build
export -f docker_push
export -f webhook
```

Additionally, testing the script logic is considerably simpler with mocking. 

{% end %}

# Pipes

While we know how to run concurrent processes, it is time to make them talk to each other. Enter pipes. 

You probably already know that `ls | grep` sends all output from `ls` straight to `grep`. But how does that work?


A typical linux process has at least three file descriptors (fd) open (unless it explicitly closes them):

* `stdin` (0 fd)
* `stdout` (1 fd)
* `stderr` (2 fd)

Pipes simply forward data between these file descriptors. In many cases, they are far more useful than files, especially since a process can detect when a pipe closes and exits gracefully. 

For better understanding, I recommend reading a [kernel code for pipes](https://github.com/torvalds/linux/blob/059dd502b263d8a4e2a84809cf1068d6a3905e6f/fs/pipe.c). But if that sounds a bit too hardcore for now, I’ve got you covered with a simple C program to illustrate the concept:

{% note(header="Pipes illustration", clickable=true, hidden = true) %}

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    int pipefd[2];
    pid_t pid1, pid2;

    // Creating a pipe
    if (pipe(pipefd) == -1) {
        perror("pipe");
        exit(EXIT_FAILURE);
    }

    // Creating a child process
    pid1 = fork();
    if (pid1 == -1) {
        perror("fork");
        exit(EXIT_FAILURE);
    }

    if (pid1 == 0) { // Child 1 (Writer)
        // Close the read end of the pipe
        close(pipefd[0]);

        // Redirect standard output to the write end of the pipe
        dup2(pipefd[1], STDOUT_FILENO);

        // Close the original write end of the pipe (important!)
        close(pipefd[1]);

        // Execute a command or perform some action that writes to stdout
        // Example:
        execlp("ls", "ls", "-l", NULL); // List files in long format
        perror("execlp"); // Only reached if execlp fails
        exit(EXIT_FAILURE);
    }

    // Create the second child process
    pid2 = fork();
    if (pid2 == -1) {
        perror("fork");
        exit(EXIT_FAILURE);
    }

    if (pid2 == 0) { // Child 2 (Reader)
        // Close the write end of the pipe
        close(pipefd[1]);

        // Redirect standard input to the read end of the pipe
        dup2(pipefd[0], STDIN_FILENO);

        // Close the original read end of the pipe (important!)
        close(pipefd[0]);

        // Execute a command or perform some action that reads from stdin
        // Example:
        execlp("wc", "wc", "-l", NULL); // Count lines from input
        perror("execlp"); // Only reached if execlp fails
        exit(EXIT_FAILURE);
    }

    // Parent process (Closes both ends of the pipe)
    close(pipefd[0]);
    close(pipefd[1]);

    // Wait for both child processes to finish
    waitpid(pid1, NULL, 0);
    waitpid(pid2, NULL, 0);

    printf("Parent process finished.\n");

    return 0;
}
```

Command to build: 
```bash
gcc filename.c -o filename
./filename
```
{% end %}

Here are a few examples of how we can write to two places from a single command: 

```bash
(echo "to_stdout"; echo "to_stderr" >&2) \
  2> >(xargs -I{} echo "from_stderr:{}") \
  1> >(xargs -I{} echo "from_stdout:{}")
```

or with `tee`

```bash
echo "image" | tee >(xargs -I{} echo "from_tee:{}") | xargs -I{} echo "from_tee2:{}"
```

Yet, it looks messy. I bet we had something involving names... Yep, they named pipes!

{% note(header="Connecting pipes and file descriptors") %}

Later, you might stumble upon something like: `exec 3>/tmp/reg1.fifo`. This command connects file descriptor `3` to a named pipe.

I didn’t add this to confuse you—it’s practical. The idea is to open the pipe just once and keep it open until we explicitly close it later.

{% end %}

```bash
# Let’s create some named pipes for our pushers
mkfifo /tmp/reg1.fifo
mkfifo /tmp/reg2.fifo

# Start the worker that builds images
(
    # Redirect file descriptors to the pipes
    exec 3>/tmp/reg1.fifo
    exec 4>/tmp/reg2.fifo

    # Build images
    for i in $LIST; do
        docker_build ${i}

        # Notify the pipes about our triumph
        echo ${i} >&3
        echo ${i} >&4
    done

    # Once all images are built, close the pipes
    exec 3>&-
    exec 4>&-
    # And clean up the mess
    rm /tmp/reg1.fifo /tmp/reg2.fifo
)& # The `&` means we're running this in the background

# Use xargs to avoid blocking the docker_build worker.
# Xargs buffers all stdin: that is how we don't block docker_build.
xargs -n1 -I{} bash -c "docker_push reg1/{}" < /tmp/reg1.fifo &
xargs -n1 -I{} bash -c "docker_push reg2/{}" < /tmp/reg2.fifo &

# Wait for all processes to complete
wait
```

If you think that **↑** was overengineering—close your eyes and wait for SIGTERM. 

# Communication to webhook

Just like there’s more than one way to skin a cat, there’s more than one way to sync processes: signals + trap, lock files, the `wait` command, or pipes.

As for me — pipes are easiest in implementation and understanding. But hey, if you’re curious about the other methods, drop a comment, and we’ll dive in! 

So, let's go piping (pipening?)

We know that if someone tries to read from a pipe no one writes, it’ll just hang. And vice versa. It’s like trying to shake hands with someone who forgot to show up.

```bash
#!/usr/bin/env bash

mkfifo /tmp/pipe
(
  echo "I'm waiting for the pipe."
  < /tmp/pipe
  echo "I'm done waiting for the pipe."
)&

sleep 1
echo "I'm going to write to the pipe."
echo > /tmp/pipe
sleep 1
echo "I'm done writing to the pipe."

# Result:
# I'm waiting for the pipe.
# I'm going to write to the pipe.
# I'm done waiting for the pipe.
# I'm done writing to the pipe.
```

See? The reader waits until the writer shows up, then they both move on with their lives. 


With two processes, things get a bit trickier:

```bash
#!/usr/bin/env bash

mkfifo /tmp/pipe1
mkfifo /tmp/pipe2
(
  < /tmp/pipe1
  < /tmp/pipe2
)&

echo > /tmp/pipe2
echo > /tmp/pipe1
```

Both processes just sit there, stubbornly waiting for the other to make a move. Classic deadlock. Reminds me of my previous marriage.

Solution? Run pipe-waiting processes in separate forks and wait for them.  

Here's how:

```bash
mkfifo /tmp/pipe1
mkfifo /tmp/pipe2
(
  cat < /tmp/pipe1 &
  pid1=$!
  cat < /tmp/pipe2 &
  pid2=$!
  wait $pid1 $pid2
  echo "I'm done with pipes!"
)&

echo > /tmp/pipe2
echo > /tmp/pipe1
```

# The result

If we combine all of this knowledge into a script, we will get something like this: 

```bash
#!/usr/bin/env bash

LIST="image1 image2 image3"

function docker_build() {
    echo Start build for $1
    sleep 1.5
    echo End build for $1
}

function docker_push() {
    echo Start push for $1
    sleep $2
    echo End push for $1
}

function webhook() {
    echo Start webhook for $1
    sleep 0.1
    echo End webhook for $1
}

export -f docker_build
export -f docker_push
export -f webhook

mkfifo /tmp/reg1.fifo
mkfifo /tmp/reg2.fifo
(
exec 3>/tmp/reg1.fifo
exec 4>/tmp/reg2.fifo
for i in $LIST; do
  mkfifo /tmp/reg1-${i}.fifo
  mkfifo /tmp/reg2-${i}.fifo
  docker_build ${i}
  (
    cat /tmp/reg1-${i}.fifo &> /dev/null &
    wait_for_reg1=$!
    cat /tmp/reg2-${i}.fifo &> /dev/null &
    wait_for_reg2=$!
    wait $wait_for_reg1 $wait_for_reg2
    webhook ${i}
    rm /tmp/reg1-${i}.fifo /tmp/reg2-${i}.fifo
  )&
  echo ${i} >&3
  echo ${i} >&4
done
exec 3>&-
exec 4>&-
rm /tmp/reg1.fifo /tmp/reg2.fifo
)&

xargs -n1 -I{} bash -c "docker_push reg1/{} 5; echo > /tmp/reg1-{}.fifo" < /tmp/reg1.fifo &
xargs -n1 -I{} bash -c "docker_push reg2/{} 1; echo > /tmp/reg2-{}.fifo" < /tmp/reg2.fifo &
wait
```

And as a bonus, here are `Go` code that are doing the same:
{% note(header="Go code", clickable=true, hidden = true) %}

```go
package main

import (
	"log"
	"math/rand"
	"sync"
	"time"
)

type wait struct {
	image string
	done  chan<- struct{}
}

func main() {
	group := sync.WaitGroup{}
	ch1 := make(chan wait, 100)
	ch2 := make(chan wait, 100)
	images := []string{"image1", "image2", "image3", "image4", "image5"}

	group.Add(1)
	go func() {
		defer group.Done()
		for _, i := range images {
			// random sleep
			wait1 := make(chan struct{})
			wait2 := make(chan struct{})
			log.Printf("Start docker build %s", i)
			time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond)
			log.Printf("Sending %s to channels", i)
			group.Add(1)
			go func() {
				defer group.Done()
				webhook_wg := sync.WaitGroup{}
				webhook_wg.Add(2)
				go func() {
					defer webhook_wg.Done()
					<-wait1
				}()
				go func() {
					defer webhook_wg.Done()
					<-wait2
				}()
				webhook_wg.Wait()
				log.Printf("Start webhook %s", i)
			}()
			ch1 <- wait{i, wait1}
			ch2 <- wait{i, wait2}
		}
		close(ch1)
		close(ch2)
	}()
	group.Add(2)
	go func() {
		defer group.Done()
		for i := range ch1 {
			log.Printf("Processing first repo %s", i.image)
			time.Sleep(time.Duration(rand.Intn(500)) * time.Millisecond)
			log.Printf("Processed %s in first repo", i.image)
			close(i.done)
		}
	}()
	go func() {
		defer group.Done()
		for i := range ch2 {
			log.Printf("Processing second repo %s", i.image)
			time.Sleep(time.Duration(rand.Intn(3200)) * time.Millisecond)
			log.Printf("Processed %s in second repo", i.image)
			close(i.done)
		}
	}()
	group.Wait()
}
```

{% end %}

This is my first post, so please leave a comment and thank you for reading. 