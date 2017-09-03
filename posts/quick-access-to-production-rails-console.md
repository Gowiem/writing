# Quick Access to Production Rails Console

## Intro

Back when I was a dev manager, I used to monotonously do the same sequence of terminal commands 5-10 times a day: `ssh` into a particular host, `sudo su` to a user with correct permissions, `cd` to a project, and then open up `rails console` or `hbase shell`. I did this so many times that it became second nature. My product guy would have a problem and I'd say "Alright, let me look at HBase real quick, we'll figure that out" or "Oh I can fix that user's account, hold on".

Being an engineer, this same repetitive motion drove me nuts and I quickly got tired of it. I set out to automate as much of the process to get to a production shell prompt as soon as possible and that's when I came across [`expect`](https://en.wikipedia.org/wiki/Expect). `expect` is a handy tool that allows you to write scripts against interactive text interfaces, which means you can automate a simple workflow like ssh'ing into a host and running commands against it. It fit perfectly and below I'll share with you an abstract version of my production rails console script and another I just developed to get into rails console in a Docker environment.

## Traditionally Hosted Scenario

Sharing the code for these scripts will help explain a lot so I'm going to start off with that:

```
#!/usr/bin/expect -f
# prod_railsc.sh <hostname - optional>

set hostname [lindex $argv 0]
set default_hostname $YOUR_PRIMARY_RAILS_HOST_HERE
set prompt "$ " # This may be $ or # or something else dependent on host environment. Change accordingly.

if {$hostname != ""} {
  spawn ssh $hostname
} else {
  spawn ssh $default_hostname
}

expect $prompt

# NOTE: You might not need this step if the user you ssh in as has Rails available.
send "sudo su $USER_WHO_HAS_ACCESS_TO_RAILS_GEMS\r"
expect $prompt

send "cd $RAILS_PROJECT_DIRECTORY\r"
expect $prompt

send "rails console\r"
expect "irb(main):001:0> "

interact
```

The above is doing what we've discussed: ssh'ing into a host, changing to a different user, changing directories, and then opening up a `rails console` session. The way this works is that `expect` creates a new process (via `spawn`) for the ssh session and then issues (`send`) commands against that ssh process, 'expecting' (`expect`) certain output from those responses. If at any point what it expects doesn't match up after a certain time frame then it'll exit. Otherwise, it proceeds through these steps and we finally get to our `interact` command which brings the spawned process to the foreground allowing us to take over the session. Ta-da, we've got a remote `rails console` and it takes only a few seconds and considerably less typing!

## Docker Hosted Scenario

The reason I got back into thinking about these expect scripts is due to Docker. I have a Dockerized Rails application that I was putting into production with AWS Elastic Beanstalk and I was updating the README with instructions on how to open rails console in that environment. While writing them I was thinking to myself "Well, this is going to be a PITA everytime", so I figured I could adapt my old `prod_railsc.sh` script to the task. This is what I ended up with:

```
#!/usr/bin/expect -f
# prod_railsc.sh <hostname - optional>

set hostname [lindex $argv 0]
set default_hostname $YOUR_EBS_HOST_HERE

if {$hostname != ""} {
  spawn ssh $hostname
} else {
  spawn ssh $default_hostname
}

expect "$"

send "sudo su - root\r"
expect "#"

set image_id_cmd "docker images -q | head -n 1"

send "$image_id_cmd\r"
expect "$image_id_cmd\r\n"
expect -re "(.*)\r\n(.*) "

set image_id $expect_out(1,string)

# Run a interactive Docker command against our image to kick off rails console.
send "docker run -it --rm $image_id rails console\r"
expect "irb(main):001:0> "

interact
```

This does everything very similar to our first script, except we have to fetch the image_id of the Rails Docker image before we can run the `rails console` command against it. `expect` allows us to use regular expressions to pull out specific output from our previous command, which we use above in the `expect -re "(.*)\r\n(.*) "` line. We then pull out the `image_id` variable and use that in our `docker run` command. This only works in Single Container nodes, but it could be adapted pretty easily if you're running other services on the same host.

## Wrapping Up

We've got ourselves a fast, automated way to access `rails console` in both traditionally hosted and Docker hosted rail's environments. If you're interacting with your Rails instance in production to pull important database information, administer data, or even kick off one-time jobs then this script can save you sometime in the long run!

If you have an improvement or a better way of accessing your production shell then I would love to hear about it in the comments!
