---
layout: post
title:  "Get PID of process run as non-root user in init.d script"
date:   Thu May 19 15:51:31 2016
categories: linux rhel centos initd processes
---
It took a while to find a reasonable solution to this problem, so here it is so I don't forget it in the future.
I wanted to create a system user with no login shell to run an application, and also capture and write it's PID to a file in `/var/run/myapp.pid`. The initial difficulty I had in figuring this out was getting the PID of the process that is launched, rather than the PID of the shell that is started to launch the process as the system user. Here is the solution I came up with:


Create the user and ownership of the files it needs access to:

{% highlight sh %}
$ adduser myappuser -r --shell=/bin/false
$ chown myappuser:myappuser -R /opt/myapp
$ chown myappuser:myappuser -R /etc/myapp
{% endhighlight %}

I suppose the chown could be done differently, but the bootstrapping of the instance copied all files as root, and I didn't want to do very many modifications to that process at this stage.

Lightly edited version of the init.d script for this service:

{% highlight sh %}
#!/bin/bash

pid_file="/var/run/myapp.pid"
name="myapp"

get_pid() {
    cat "$pid_file"
}

is_running() {
    [ -f "$pid_file" ] && ps `get_pid` > /dev/null 2>&1
}

case $1 in
    start)
        if is_running; then
            echo "Already started"
        else
            echo "Starting $name"
            touch $pid_file
            chown myappuser:myappuser $pid_file
            PID=$(su myappuser -c '/opt/myapp/bin/myapp \
            1>> /var/log/myapp.log 2>> /var/log/myapp.err & echo ${!}')

            echo $PID > $pid_file
        fi
        ;;
     stop)
        if is_running; then
            echo -n "Stopping $name.."
            kill `get_pid`

        for i in {1..10}
            do
                if ! is_running; then
                    break
                fi

                echo -n "."
                sleep 1
            done
            echo
        if is_running; then
                    echo "Not stopped; may still be shutting down or shutdown may have failed"
                    exit 1
                else
                    echo "Stopped"
                    if [ -f "$pid_file" ]; then
                        rm "$pid_file"
                    fi
                fi
            else
                echo "Not running"
        fi
         ;;
     restart)
        if is_running; then
            $0 stop
        else
            echo "Service was not running. Starting..."
        fi
        $0 start
        ;;
     status)
        if is_running; then
           echo "Running"
        else
           echo "Stopped"
           exit 1
        fi
        ;;
     *)
        echo "usage: review {start|stop|restart|status}" ;;
 esac
 exit 0
{% endhighlight %}
