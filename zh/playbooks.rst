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


`vars` 部分包括可用于play中的一组变量/值，如下所示  ::

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

每个play包括一系列任务。任务在和主机模式匹配的所有主机上，按顺序一次一个执行，
然后再执行下一个任务。在一个play中，所有主机会得到同样的任务指令。play的目的
就是把选择的主机和任务对应起来。

从上到下执行playbook时，有失败任务的主机会被从整个playbook的循环中剔除。如果出错的话，就更改playbook文件、重新运行。

每个任务的目标是以特定的参数执行模块。前面所说的变量，可作为模块的参数使用。

模块是 'idempotent'，如果你重新运行，它们会做出所需改变，使系统成为所需的状态。

如果是chmod或setsebool这样的命令， `command` 和 `shell` 模块通常会再次运行这些命令。有个'creates'标志可使这些模块成为 idempotent。

每个任务都应有个 `name` ，运行playbook时会包括在其输出中。名称最好对任务步骤有
较好的描述。如果未提供name，赋给action的字符串会用于输出。

一个简单的任务如下所示，大多数模块像service模块一样，接受键=值参数::

   tasks:
     - name: make sure apache is running
       action: service name=httpd state=running

 `command` 和 `shell` 模块不使用键=值的形式，接受参数列表。这使它们如你期望的。很简单::

   tasks:
     - name: disable selinux
       action: command /sbin/setenforce 0

command和shell模块关心返回值，如果你有成功退出返回值非0的命令，可以这么做::

   tasks:
     - name: run this command and ignore the result
       action: shell /usr/bin/somecommand & /bin/true

action行中可使用变量。假设你在vars一节定义一个vhost变量，可以这么做::

   tasks:
     - name: create a virtual host file for $vhost
       action: template src=somefile.j2 dest=/etc/httpd/conf.d/$vhost

同样的变量可用在模板中，我们稍后再讨论。

在很基本的playbook中，全部任务会直接在play中列出；正常情况下，可以用'include:'指令对任务进行分解。这个我们稍后再谈。

在发生变化时执行操作
````````````````````````````

如上所述，模块是idempotent，它们在远程系统做出变化后可以relay。以此为基础，playbooks有一个基本的事件系统，可用来应对变化。

这些notify操作在每个play结束时激活，而且只激活一次。例如，可能有多个资源表示apache需重启，不过它只会重启一次。

在下例中，文件内容发生改变会重启两个服务，只有文件确实发生变化::

   - name: template configuration file
     action: template src=template.j2 dest=/etc/foo.conf
     notify:
        - restart memcached
        - restart apache

notify一节中列出的操作称为handler（处理程序）。

Handler(处理程序)是一系列任务，和通常的任务没什么不同，也是用名称引用。通知程序
通知handler。如果handler没人通知，它就不会运行。不管有多少东西通知handler，
它只会在特定play中所有任务运行完成后，运行一次。

下面是一handlers(处理程序)示例::

    handlers:
        - name: restart memcached
          action: service name=memcached state=restarted
        - name: restart apache
          action: service name=apache state=restarted

Handlers(处理程序)最适合用于重启服务、主机。别的场合很少用到。

.. 注::
   通知处理程序总是以编写的顺序执行。


包含文件鼓励重用
```````````````````````````````````

假如你希望在不同的play或playbook中重用一系列任务。可用包含文件(include files)来实现。使用包含的任务列表，是定义系统要实现的角色的一种很好的方式。别忘了playbook中play的目标是把一组系统对应到几个角色。下面让我们来看看是什么样子...

任务包含文件只是一个简单的任务列表，如下所示: ::

    ---
    # possibly saved as tasks/foo.yml
    - name: placeholder foo
      action: command /bin/foo
    - name: placeholder bar
      action: command /bin/bar

包含指令可以和playbook中正常的任务混合使用: ::

   tasks:
    - include: tasks/foo.yml

也可向包含中传入变量。这称为"参数化包含"。

例如，部署多个wordpress实例时，可在一单独的wordpress.yml中包括所有的wordpress任务： ::

   tasks:
     - include: wordpress.yml user=timmy
     - include: wordpress.yml user=alice
     - include: wordpress.yml user=bob

传入的变量然后就可用在包含文件里。用下面的方式引用 ::

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


