---
title: "Open sourcing Workshops-on-Demand - Part 5: Create a workshop"
date: 2023-07-24T10:43:53.548Z
author: Frederic Passeron
authorimage: /img/frederic-passeron-hpedev-192.jpg
disable: false
---
I﻿n this new article that is part of this series dedicated to the [open sourcing of our Workshops-on-Demand project](https://developer.hpe.com/blog/willing-to-build-up-your-own-workshops-on-demand-infrastructure/), I will focus on the necessary steps to build up a new workshop. I already covered most of the infrastructure part that supports the workshops. It now makes sense to emphasize a bit on the content creation.

# O﻿verview

H﻿ere is a simple flowchart describing the 10000 feet view of the creation process:

![](/img/wod-b-process.png "Workshop's creation flow.")

A﻿s you can see, no rocket science here. Just common sense. Depending on the workshop you wish to create, some obvious requirements should show up. A workshop based on a programmatic language for instance, will require the relevant kernel to be setup on the JupyterHub server. The following [page](https://gist.github.com/chronitis/682c4e0d9f663e85e3d87e97cd7d1624) will list all available kernels.

S﻿ome other workshops might need a proper infrastructure to run on. A kubernetes101 workshop fro instance could not exist without the presence of a proper Kubernetes cluster. Same thing goes for any HPE related solutions.

F﻿rom an infrastructure standpoint, a minimum of environments are necessary. You will need a test/ dev,  a staging and at least one production environment. The HPE Developer Community actually started with only a test/dev/staging on one side and a production on the other side.

I﻿ will not focus here on the first steps. I leave it to you to figure out new subjects. I will talk a bit again about the infrastructure and especially the dedicated scripts and variables that you need to create to support the lifecycle of the workshop. As usual, there are two sides to the workshop's creation. What should be done on the backend and what needs to be done on the frontend (web portal and database server mainly)

![](/img/wod-blogserie3-archi3.png "WOD Overview.")

I﻿ will consider two scenarios going further. In the first one, I will create a simple workshop that does not require any infrastructure but the JupyterHub itself. Then, in a second phase, I will go through a more complex workshop creation process that will cover most of the possible cases I could think of.

# S﻿imple workshop example:

l﻿et's consider that I plan to create a new workshop on the Go language. It becomes more and more popular among the developer community I interact with and one of the the developer was kind enough to agree with working with me on creating this new workshop. After a first meeting, where I explainied to him the creation process, and the expectations, we quickly started to work together. We defined title, abstract,  notebooks' folder name, and student range. As far as infrastructure's requirements, a new kernel was needed. 

A﻿s an admin of the Workshops-on-demand infrastructure, I had to perform several tasks:

1. ###### Test and validate installation of the new kernel on the staging backend server by:

* Creating a new branch for this test
* M﻿odifying the [backend server installation yaml file ](https://github.com/Workshops-on-Demand/wod-backend/blob/main/ansible/install_backend.yml#L326)to include the new kernel.

```
    - name: Ensure all required RPM packages are installed
      become: yes
      become_user: root
      package:
        pkg:
        - python3-devel
        - openldap-clients
        - glibc-devel
        - java
        - nfs-utils
        - zeromq-devel
        - perl-Crypt-PasswdMD5
        - openmpi-devel
        - fail2ban-all
        - perl-Proc-ProcessTable
        - perl-Data-Dumper
        - psacct
        - golang
        state: present

    - name: Ensure Go kernel is installed
      shell: env GO111MODULE=off go get -d -u github.com/gopherdata/gophernotes && cd "{{ ansible_env.HOME }}/go/src/github.com/gopherdata/gophernotes" && env GO111MODULE=off go install && mkdir -p {{ JPHUB }}/share/jupyter/kernels/gophernotes && cp kernel/* {{ JPHUB }}/share/jupyter/kernels/gophernotes

    - name: Ensure required directories under /usr/local are owned by "{{ WODUSER }}"
      become: yes
      become_user: root
      file:
        path: "{{ item }}"
        state: directory
        owner: "{{ WODUSER }}"
        group: "{{ WODUSER }}"
        mode: 0755
      with_items:
        - /usr/local/bin
        - /usr/local/share

    - name: Make gophernotes available under /usr/local/bin
      become: yes
      become_user: root
      copy:
        src: "{{ ansible_env.HOME }}/go/bin/gophernotes"
        dest: /usr/local/bin/gophernotes
        owner: "{{ WODUSER }}"
        group: "{{ WODUSER }}"
        mode: 0755

    - name: Cleanup intermediate go directory
      become: yes
      become_user: root
      file:
        path: '{{ ansible_env.HOME }}/go'
        state: absent
```

﻿* Validating the changes by testing a new backend insatll process.*
﻿ 
2﻿. 

In order to exist, a workshop requires serveral things:

From a frontend standpoint:

I﻿n the Workshops table:

A﻿ new entry will need the following:

![](/img/wod-db-entry1.png "Workshop's fields in the Database.")

**An id:** A workshop id to be used by backend server automation and Replays table to reference the associated replay video of the workshop.

**A﻿ name:** The workshop's name as it will will be displayed on the registration portal.

**A name of the folder** containing all the workshop's notebooks

**A﻿ description / abstract**

**A﻿ capacity:** The number of maximum concurrent students allowed to take on the workshop.

**A﻿ student range:** The range between which students get picked at registration time.

**R﻿eset and ldap** entries are to be used by backend server automation if dedicated reset scripts and ldap authentication are required by the workshop.

**A﻿ session type:** Workshops-on-Demand by default

**A﻿ location:** If your setup includes multiple production sites, use this field to allocate workshops according to your needs. In the case of the HPE Developer Community, some workshops can only run on a HPE GreenLake cloud environment. As a consequence, the location is set to greenlake in this case.

**A﻿vatar, role and replayLink** are superseeded by entries in the replay table. I will explain later.

![](/img/wod-db-entry2.png "Workshop's fields in the Database #2.")

**Compile:** This entry will be filled with the name of a script to be compiled at deployment time. This feature allows for instance the admin to hide login scripts and credentials in non-editable executable files.

**Varpass:**  This defines whethere a workshop require some password variable to be leveraged or not.

**R﻿eplayId:** This entry links the dedicated replay video to the workshop. it enables the presence of the replay in the learn more page of the workshop.

**W﻿orkshopImg:** As part of the lifecycle of the workshop, several emails are sent to the student. A workshop image is embbeded in the first emails.

**B﻿adgeImg:** As part of the lifecycle of the workshop, several emails are sent to the student. In the final email, a badge is included. It allows the student to share its accomplishment on SoME like linkedin for instance.

***N﻿ote:***

B﻿oth W﻿orkshopImg and B﻿adgeImg are located on the same remote web server.

**B﻿eta:** Not implemented yet :-)

**Category:** The workshops' registration portal proposes several filters to display the catlog's content. You can view all workshops, the most poular ones, or by category. Use this field to sort workshops accordingly.

**A﻿lternateLocation:** Not implemented yet. The purpose is allow automation of the relocation of a workshop in case of primary location's failure.

**D﻿uration:** All workshops are time bombed. You will define here the time alloacted to perform the workshop.

I﻿f you feel you need more details about the registration process, please take a look at the **Register Phase** paragraph in [the following introductionary blog](https://developer.hpe.com/blog/willing-to-build-up-your-own-workshops-on-demand-infrastructure/).

A﻿ set of notebooks that will be used by the student to follow instructions cells in markdown and run code cells leveraging the relevant kernel. If you are not familiar with Jupyter notebooks, a simple [101 workshop](https://developer.hpe.com/hackshack/workshop/25) is available in our Workshops-on-Demand 's catalog.

O﻿ptional:

You should now have a better understanding of the maintenance tasks associated to the backend server. Similar actions are available for the other components of the project. Checking tasks have been created for the frontend and api-db server. Having now mostly covered all the subjects related to the backend server from an infrastructure standpoint, it is high time to discuss the content part. In my next blog, I plan to describe the workshop creation process.  Time to understand how to build up some content for the JupyterHub server!

If we can be of any help in clarifying any of this, please reach out to us on [Slack](https://slack.hpedev.io/). Please be sure to check back at [HPE DEV](https://developer.hpe.com/blog) for a follow up on this. Also, don't forget to check out also the Hack Shack for new [workshops](https://developer.hpe.com/hackshack/workshops)! Willing to collaborate with us? Contact us so we can build more workshops!