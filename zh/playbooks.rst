Playbooks
=========

.. contents::
   :depth: 2
   :backlinks: top

简介
````````````

和任务执行模式相比，Playbooks是个使用ansible完全不同的方法，而且特别强大。简单地说，playbooks是个非常简单的配置管理系统、多机部署系统的基础；和现有类似系统不同，非常适合部署复杂的应用。

Playbooks可以声明配置，也可以协调任何手工有序过程的步骤，即便不同的步骤需要在一系列主机上以特别的顺序执行。它还可以同步或异步地启动任务。

对于一般任务可以直接运行 /usr/bin/ansible，playbooks更有可能存储在源码控制系统里，用于推出配置、确保远程系统的配置正确。

现在让我们深入地看看playbook是怎么工作的。可以在另一个tab中打开 `github示例目录 <https://github.com/ansible/ansible/tree/devel/examples/playbooks>`_ ，边看边做。 

Playbook语言示例
`````````````````````````

Playbooks用YAML格式表示，只有很少的语法。每个playbook由一或多个play组成。

Play的目标是把一组主机和明确定义的角色对应起来，而角色则由ansible里的任务(task)代表。在基础层面，任务只不过是对一个ansible模块的调用，在前面的章节里你可能已学过。

通过用多个play组成一个playbook，就能协调多机部署，例如：在webservers组中的所有主机上运行某些步骤，然后在数据库主机组上运行另一些步骤，然后再回来在webservers组上运行更多命令等。

下面是一个只有一个play组成的playbook::

    ---
    - hosts: webservers
      vars:
        http_port: 80
        max_clients: 200
      user: root
      tasks:
      - name: ensure apache is at the latest version
        action: yum pkg=httpd state=latest
      - name: write the apache config file
        action: template src=/srv/httpd.j2 dest=/etc/httpd.conf
        notify:
        - restart apache
      - name: ensure apache is running
        action: service name=httpd state=started
      handlers:
        - name: restart apache
          action: service name=apache state=restarted

下面我们一个一个地看playbook语言的各种特点。

基础
``````

Hosts and Users (主机与用户)
+++++++++++++++++++++++++++++

对playbook中的每个play，你都可以选择你基础设施中的目标主机，以及执行任务的远程用户。


 `hosts` 行是由冒号分割的一或多个主机组、或主机模式，详见 :ref:`patterns` 文档。
 `user` 是用户帐户名称。

    ---
    - hosts: webservers
      user: root

也可以用sudo运行命令::

    ---
    - hosts: webservers
      user: yourname
      sudo: True

还可以一个用户登录，然后sudo成root外的其它用户::

    ---
    - hosts: webservers
      user: yourname
      sudo: True
      sudo_user: postgres

如果需要为sudo指定密码，可用 `ansible-playbook` 的 ``--ask-sudo-pass`` (`-K`) 选项。
在运行sudo playbook时遇到无反应的情况，很可能是停在执行sudo时需输入密码的地方。
输入 `Control-C` 取消此playbook，然后用 `-K` 选项重新运行。

.. 重要::

    用 `sudo_user` 指令，sudo到root之外的用户时，模块参数会被暂时写到/tmp
    目录中的一个临时文件中，命令执行后会马上删除。这种情况只会在从用户 bob
    sudo 到 用户 timmy 时发生，从 bob sudo 为root，或直接以bob或root登录则
    不会出现这种情况。这些数据会暂时可读(不可写)，若要避免这种情形，请勿使用
    `sudo_user` 指令传输未加密密码。其它情况下不使用/tmp目录，没有这个问题。
    Ansible也不会在日志中记录密码参数。
   
Vars (变量) 
++++++++++++

 `vars` 部分包括可用于play中的一组变量/值，如下所示 ::

    ---
    - hosts: webservers
      user: root
      vars:
         http_port: 80
         van_halen_port: 5150
         other: 'magic'

这些变量在playbook中可以如下方式使用 ::

    $varname 或 ${varname}

如要以类似 ${other}_some_string 这样的方式使用变量，可使用后一种方式。

在模板中可使用 `Jinja2 <http://jinja.pocoo.org/docs/>`_ 模板语言的全部功能，变量的引用如下所示 :: 

    {{ varname }}

希望使用高级模板功能的用户，Jinja2 文档提供了关于构建循环、条件语句的有关信息。
这些是可选的，$varname 格式在模板文件仍然有效。

如果有关于系统发现的变量，称为 facts (事实)，这些变量会返回playbook中，在每个
系统上可像明确设置的变量一样使用。Ansible提供了几个这种变量, 变量名以ansible开头，详见 :ref:`setup` 模块文档。此外，如果安装了 ohai 和 facter，可以用它们来收集facts。Facter变量以 ``facter_`` 为前缀，Ohai变量以 ``ohai_`` 为前缀。

比如说我想把主机名写入/etc/motd文件，就可以 ::

   - name: write the motd
     action: template src=/srv/templates/motd.j2 dest=/etc/motd

在/srv/templates/motd.j2文件中::

   You are logged into {{ facter_hostname }}

现在说这些有点儿为时过早。我们来谈谈任务。

任务清单
++++++++++

Each play contains a list of tasks.  Tasks are executed in order, one
at a time, against all machines matched by the host pattern,
before moving on to the next task.  It is important to understand that, within a play,
all hosts are going to get the same task directives.  It is the purpose of a play to map
a selection of hosts to tasks.

When running the playbook, which runs top to bottom, hosts with failed tasks are
taken out of the rotation for the entire playbook.  If things fail, simply correct the playbook file and rerun.

The goal of each task is to execute a module, with very specific arguments.
Variables, as mentioned above, can be used in arguments to modules.

Modules are 'idempotent', meaning if you run them
again, they will make the changes they are told to make to bring the
system to the desired state.  This makes it very safe to rerun
the same playbook multiple times.  They won't change things
unless they have to change things.

The `command` and `shell` modules will typically rerun the same command again,
which is totally ok if the command is something like
'chmod' or 'setsebool', etc.  Though there is a 'creates' flag available which can
be used to make these modules also idempotent.

Every task should have a `name`, which is included in the output from
running the playbook.   This is output for humans, so it is
nice to have reasonably good descriptions of each task step.  If the name
is not provided though, the string fed to 'action' will be used for
output.

Here is what a basic task looks like, as with most modules,
the service module takes key=value arguments::

   tasks:
     - name: make sure apache is running
       action: service name=httpd state=running

The `command` and `shell` modules are the one modules that just takes a list
of arguments, and don't use the key=value form.  This makes
them work just like you would expect. Simple::

   tasks:
     - name: disable selinux
       action: command /sbin/setenforce 0

The command and shell module care about return codes, so if you have a command
who's successful exit code is not zero, you may wish to do this::

   tasks:
     - name: run this command and ignore the result
       action: shell /usr/bin/somecommand & /bin/true

Variables can be used in action lines.   Suppose you defined
a variable called 'vhost' in the 'vars' section, you could do this::

   tasks:
     - name: create a virtual host file for $vhost
       action: template src=somefile.j2 dest=/etc/httpd/conf.d/$vhost

Those same variables are usable in templates, which we'll get to later.

Now in a very basic playbook all the tasks will be listed directly in that play, though it will usually
make more sense to break up tasks using the 'include:' directive.  We'll show that a bit later.

Running Operations On Change
````````````````````````````

As we've mentioned, modules are written to be 'idempotent' and can relay  when
they have made a change on the remote system.   Playbooks recognize this and
have a basic event system that can be used to respond to change.

These 'notify' actions are triggered at the end of each 'play' in a playbook, and
trigger only once each.  For instance, multiple resources may indicate
that apache needs to be restarted, but apache will only be bounced once.

Here's an example of restarting two services when the contents of a file
change, but only if the file changes::

   - name: template configuration file
     action: template src=template.j2 dest=/etc/foo.conf
     notify:
        - restart memcached
        - restart apache

The things listed in the 'notify' section of a task are called
handlers.

Handlers are lists of tasks, not really any different from regular
tasks, that are referenced by name.  Handlers are what notifiers
notify.  If nothing notifies a handler, it will not run.  Regardless
of how many things notify a handler, it will run only once, after all
of the tasks complete in a particular play.

Here's an example handlers section::

    handlers:
        - name: restart memcached
          action: service name=memcached state=restarted
        - name: restart apache
          action: service name=apache state=restarted

Handlers are best used to restart services and trigger reboots.  You probably
won't need them for much else.

.. note::
   Notify handlers are always run in the order written.


Include Files And Encouraging Reuse
```````````````````````````````````

Suppose you want to reuse lists of tasks between plays or playbooks.  You can use
include files to do this.  Use of included task lists is a great way to define a role
that system is going to fulfill.  Remember, the goal of a play in a playbook is to map
a group of systems into multiple roles.  Let's see what this looks like...

A task include file simply contains a flat list of tasks, like so::

    ---
    # possibly saved as tasks/foo.yml
    - name: placeholder foo
      action: command /bin/foo
    - name: placeholder bar
      action: command /bin/bar

Include directives look like this, and can be mixed in with regular tasks in a playbook::

   tasks:
    - include: tasks/foo.yml

You can also pass variables into includes.  We call this a 'parameterized include'.

For instance, if deploying multiple wordpress instances, I could
contain all of my wordpress tasks in a single wordpress.yml file, and use it like so::

   tasks:
     - include: wordpress.yml user=timmy
     - include: wordpress.yml user=alice
     - include: wordpress.yml user=bob

Variables passed in can then be used in the included files.  You can reference them like this::

   $user

(In addition to the explicitly passed in parameters, all variables from
the vars section are also available for use here as well.)

.. note::
   Task include statements are only usable one-level deep.
   This means task includes can not include other
   task includes.  This may change in a later release.

Includes can also be used in the 'handlers' section, for instance, if you
want to define how to restart apache, you only have to do that once for all
of your playbooks.  You might make a handlers.yml that looks like::

   ----
   # this might be in a file like handlers/handlers.yml
   - name: restart apache
     action: service name=apache state=restarted

And in your main playbook file, just include it like so, at the bottom
of a play::

   handlers:
     - include: handlers/handlers.yml

You can mix in includes along with your regular non-included tasks and handlers.

Includes can also be used to import one playbook file into another. This allows
you to define a top-level playbook that is composed of other playbooks.

For example::

    - name: this is a play at the top level of a file
      hosts: all
      user: root
      tasks:
      - name: say hi
        tags: foo
        action: shell echo "hi..."

    - include: load_balancers.yml
    - include: webservers.yml
    - include: dbservers.yml

Note that you cannot do variable substitution when including one playbook
inside another.

.. note::

   You can not conditionally path the location to an include file,
   like you can with 'vars_files'.  If you find yourself needing to do
   this, consider how you can restructure your playbook to be more
   class/role oriented.  This is to say you cannot use a 'fact' to
   decide what include file to use.  All hosts contained within the
   play are going to get the same tasks.  ('only_if' provides some
   ability for hosts to conditionally skip tasks).

Executing A Playbook
````````````````````

Now that you've learned playbook syntax, how do you run a playbook?  It's simple.
Let's run a playbook using a parallelism level of 10::

    ansible-playbook playbook.yml -f 10

Tips and Tricks
```````````````

Look at the bottom of the playbook execution for a summary of the nodes that were executed
and how they performed.   General failures and fatal "unreachable" communication attempts are
kept seperate in the counts.

If you ever want to see detailed output from successful modules as well as unsuccessful ones,
use the '--verbose' flag.  This is available in Ansible 0.5 and later.

Also, in version 0.5 and later, Ansible playbook output is vastly upgraded if the cowsay
package is installed.  Try it!

In version 0.7 and later, to see what hosts would be affected by a playbook before you run it, you
can do this::

    ansible-playbook playbook.yml --list-hosts.

.. seealso::

   :doc:`YAMLSyntax`
       Learn about YAML syntax
   :doc:`playbooks`
       Review the basic Playbook language features
   :doc:`playbooks2`
       Learn about Advanced Playbook Features
   :doc:`bestpractices`
       Various tips about managing playbooks in the real world
   :doc:`modules`
       Learn about available modules
   :doc:`moduledev`
       Learn how to extend Ansible by writing your own modules
   :doc:`patterns`
       Learn about how to select hosts
   `Github examples directory <https://github.com/ansible/ansible/tree/devel/examples/playbooks>`_
       Complete playbook files from the github project source
   `Mailing List <http://groups.google.com/group/ansible-project>`_
       Questions? Help? Ideas?  Stop by the list on Google Groups


