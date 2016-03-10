# Deployment of Grimoire Open Development Analytics platform for The Document Foundation

## 1. Introduction

This brief document explains how to deploy the Grimoire Open Development Analytics platform for [The Document Foundation](https://www.documentfoundation.org/). These are the main steps included in the following sections:
* Installation of the docker container
* Data retrieval
* Data enrichment
* Publication on a Kibana dashboard customized by Bitergia

## 2. Requirements

* The host machine should be a GNU/Linux box with at least: git, ssh, docker(>=1.5) and docker-compose (>= 1.5)
* You must have a gerrit account in gerrit.libreoffice.org with your ssh public key added to it.
* 50 GB hard disk free space for git repositories.
* Chrome/Chromium is the web browser recommended for performance.

## 3. Install the Grimore docker container

The installation of the docker container is pretty straightforward:

* Install [docker](https://docs.docker.com/engine/installation/) and [docker-compose](https://docs.docker.com/compose/install/) in your host machine
* Create a user with login _gelk_ and add it to the _docker_ group
* Add _gelk_ user to docker group
<pre>
root@dellx:~# adduser --disabled-password gelk
root@dellx:~# adduser gelk docker
</pre>
* Login as _gelk_ user
<pre>
gelk@dellx:~$ id
uid=1002(gelk) gid=1002(gelk) grupos=1002(gelk),132(docker)
</pre>
* Create a directory "/home/gelk/devel"
<pre>
gelk@dellx:~$ mkdir ~/devel
</pre>
* Clone GrimoireELK
<pre>
gelk@dellx:~$ git clone https://github.com/grimoirelab/GrimoireELK.git devel/GrimoireELK
</pre>
* Create ElasticSearch data dir for persistence::
<pre>
gelk@dellx:~$ mkdir -p ~/Docker/data/elasticsearch/
</pre>
* Start BiDEK docker compose:
<pre>
gelk@dellx:~$ DATA_DOCKER=~/Docker/data docker-compose -f devel/GrimoireELK/docker/compose/bidek.yml up
Starting compose_redis_1
Starting compose_elasticsearch_1
Starting compose_kibiter_1
Starting compose_kibiter-edit_1
Starting compose_mariadbdata_1
Starting compose_mariadb_1
Starting compose_gelk_1
</pre>

All the logs from all the containers will be printed in this console, so open a new one to start working with the platform.

## 4. Retrieve TDF data sources

Enter the docker gelk container:

<pre>
gelk@dellx:~$ docker exec -t -i compose_gelk_1 env TERM=xterm /bin/bash
</pre>

Now, we will start with the data retrieval. One tip before going on, for testing purpose you can add the "--from-date"  paramater to the command "p2o.py", it will allow you to download just a slice of the data. For instance, from January 1st 2016 you should include: `--from-date "2016-01-01"`

### 4.1 Bugzilla

This process takes around 11 hours to finish. It is recommended to execute it inside a screen/tmux.

```
bitergia@a27769af72e3:~$ cd GrimoireELK/utils/
bitergia@89a865f18523:~/GrimoireELK/utils$ ./p2o.py -e http://elasticsearch:9200 -g  bugzilla  https://bugs.documentfoundation.org > ~/TDF-bugzilla.log 2>&1
```

To test the process, you can get the bugs since March 2016:

<pre>
bitergia@a27769af72e3:~/GrimoireELK/utils$ ./p2o.py -e http://elasticsearch:9200 -g  bugzilla  https://bugs.documentfoundation.org --from-date "2016-03-01"
</pre>

Once the process is finished you can check the number of bugs downloaded in the log file or using the Elastic Search database with the following command:

<pre>
bitergia@a27769af72e3:~/GrimoireELK/utils$ curl -s http://elasticsearch:9200/bugzilla_https:__bugs.documentfoundation.org/_search | python -m json.tool | grep '"total": '
</pre>

### 4.2 Gerrit

First you need an account for the code review plataform at [gerrit.libreoffice.org](gerrit.libreoffice.org). Ensure your account has a [public SSH key](https://gerrit.libreoffice.org/#/settings/ssh-keys) and the typical user name. Before copying and pasting the content below, replace GERRIT_USER with your user name.

Next step is to copy you SSH public key to the docker instance. In our case, we have it in the host machine (which is 172.17.42.1) for the account "acs".

<pre>
bitergia@a27769af72e3:~$ scp -r acs@172.17.42.1:.ssh ~/
bitergia@a27769af72e3:~$ eval `ssh-agent -s`
bitergia@a27769af72e3:~$ ssh-add ~/.ssh/id_rsa
</pre>

At this point, your public key is both copied to the container and set up in gerrit.libreoffice.org.

In order to check that the gerrit user config is working, execute the command (remember to replace GERRIT_USER with your username):

<pre>
bitergia@a27769af72e3:~/GrimoireELK/utils$ ssh -p 29418 GERRIT_USER@gerrit.libreoffice.org gerrit  version
gerrit version 2.11.7
</pre>

Time to start retrieving code reviews. Use the commands below (again, with you gerrit account):

<pre>
bitergia@89a865f18523:~/GrimoireELK/utils$ ./p2o.py -e http://elasticsearch:9200 -g gerrit --user GERRIT_USER --url gerrit.libreoffice.org > ~/TDF-gerrit-all.log 2>&1
</pre>

Once the process finishes you can check the number of reviews downloaded in the log file and again using the Elastic Search database with this command:

<pre>
bitergia@a27769af72e3:~/GrimoireELK/utils$ curl -s http://elasticsearch:9200/gerrit_gerrit.libreoffice.org/_search | python -m json.tool | grep '"total": '
</pre>

### 4.3 Git:

You will need your gerrit user account configured as explained in the previous Gerrit section. Don't underestimate the space, data will need around 50 GB of free space to clone all git repositories to be analyzed.

Start the Git data retrieval with the following command (remeber to replace the GERRIT_USER with your gerrit user name):
<pre>
bitergia@89a865f18523:~/GrimoireELK/utils$ ssh -p 29418 GERRIT_USER@gerrit.libreoffice.org gerrit ls-projects | awk '{print "./p2o.py -e http://elasticsearch:9200 --index git_TDF -g git git://gerrit.libreoffice.org/"$1}' | sh > ~/TDF-git-all.log 2>&1
</pre>

_Tip_: For testing purposes, you may want to get only the commits produced in 2016 with a command like this one:
<pre>
ssh -p 29418 lcanas@gerrit.libreoffice.org gerrit ls-projects | awk '{print "./p2o.py -e http://elasticsearch:9200 --index git_TDF -g git git://gerrit.libreoffice.org/"$1" --from-date \"2016-01-01\""}'
</pre>

Once the process finish you can check the number of bugs downloaded in the log file and also from Elastic Search, with the command:

<pre>
bitergia@a27769af72e3:~/GrimoireELK/utils$ curl -s http://elasticsearch:9200/git_tdf/_search | python -m json.tool | grep '"total": '
</pre>

## 5. Updating TDF data sources

The update process of the data sources uses the same commands shown above. Our recommendation is to change the output log files so you can analyze them later. An initial approach to have periodic updates is to use linux crontab to schedule them. Next versions of the product will include an scheduler.

## 6. Index enrichment

Before showing the data on the Kibana dashboards, the raw index created above should be enriched.

The commands to enrich bugzilla and gerrit are the same but with the option "--enrich_only". For git the command is also the same, but only applied to the global "git_TDF" index.

<pre>
bitergia@89a865f18523:~/GrimoireELK/utils$ ./p2o.py -e http://elasticsearch:9200 -g --enrich_only bugzilla  https://bugs.documentfoundation.org > ~/TDF-bugzilla-enrich.log 2>&1
bitergia@89a865f18523:~/GrimoireELK/utils$ ./p2o.py -e http://elasticsearch:9200 -g --enrich_only gerrit --user GERRIT_USER --url gerrit.libreoffice.org > ~/TDF-gerrit-all-enrich.log 2>&1
bitergia@89a865f18523:~/GrimoireELK/utils$ ./p2o.py -e http://elasticsearch:9200 -g --enrich_only --index git_TDF git '' > ~/TDF-git-all-enrich.log 2>&1
</pre>

## 7. Kibana dashboards creation

### 7.1 Dashboards templates import

Each data source has its own dashboard template, in order to import them into Kibana, execute the following commands:

<pre>
bitergia@a27769af72e3:~/GrimoireELK/utils$ ./kidash.py -e http://elasticsearch:9200 --import ../dashboards/git-activity.json
bitergia@a27769af72e3:~/GrimoireELK/utils$ ./kidash.py -e http://elasticsearch:9200 --import ../dashboards/gerrit-activity.json
bitergia@a27769af72e3:~/GrimoireELK/utils$ ./kidash.py -e http://elasticsearch:9200 --import ../dashboards/bugzilla-testing.json
bitergia@a27769af72e3:~/GrimoireELK/utils$ ./kidash.py -e http://elasticsearch:9200 --list
http://elasticsearch:9200/.kibana/dashboard/_search?size=10000
Git-Activity
Gerrit-Activity
BugsMozilla
</pre>

### 7.2 Dashboards creation

We are getting closer. Now we'll use "e2k.py" to let Kibana knows about the data:

```
bitergia@a27769af72e3:~/GrimoireELK/utils$ ./e2k.py -e http://elasticsearch:9200 -i git_tdf_enrich -d Git-Activity
bitergia@a27769af72e3:~/GrimoireELK/utils$ ./e2k.py -e http://elasticsearch:9200 -i gerrit_gerrit.libreoffice.org_enrich -d Gerrit-Activity
bitergia@a27769af72e3:~/GrimoireELK/utils$ ./e2k.py -e http://elasticsearch:9200 -i bugzilla_https:__bugs.documentfoundation.org_enrich -d BugsMozilla
```

Now the information is almost ready. We only need to set up a default index for Kibana.

* Visit this link: http://localhost:5601/app/kibana#/dashboard/BugsMozilla__bugzilla_https:__bugs.documentfoundation.org_enrich
* Click on the left on the Index named gerrit_gerrit.libreoffice.org_enrich (you could use any of the three we just created)
![Image of Kibana](https://raw.githubusercontent.com/grimoirelab/use_cases/master/documentfoundation/img/screenshot01e.png)
* Click on the green star to make it the default index
![Image of Kibana](https://raw.githubusercontent.com/grimoirelab/use_cases/master/documentfoundation/img/screenshot02e.png)
* And .. now we are ready to start playing with Kibana.

_Tip_: The first time you see the dashboard, pay attention to the time frame displayed, sometimes it is set up to show the last 15 minutes and you may not have anything to be shown.

Enjoy them!:
* [http://localhost:5601/app/kibana#/dashboard/BugsMozilla__bugzilla_https:__bugs.documentfoundation.org_enrich](http://localhost:5601/app/kibana#/dashboard/BugsMozilla__bugzilla_https:__bugs.documentfoundation.org_enrich)
* [http://localhost:5601/app/kibana#/dashboard/Gerrit-Activity__gerrit_gerrit.libreoffice.org_enrich](http://localhost:5601/app/kibana#/dashboard/Gerrit-Activity__gerrit_gerrit.libreoffice.org_enrich)
* [http://localhost:5601/app/kibana#/dashboard/Git-Activity__git_tdf_enrich](http://localhost:5601/app/kibana#/dashboard/Git-Activity__git_tdf_enrich)

### Metadashboard

There is one way to have the different dashboards linked from a Kibana instance, if you want to get that you'll have to execute the commands below which add some links:

```
bitergia@a27769af72e3:~/GrimoireELK/utils$
curl -XPOST "http://elasticsearch:9200/.kibana/metadashboard/main" -d'
{
    "Bugzilla":"BugsMozilla__bugzilla_https:__bugs.documentfoundation.org_enrich",
    "Git":"Git-Activity__git_tdf_enrich",
    "Gerrit":"Gerrit-Activity__gerrit_gerrit.libreoffice.org_enrich"
}'
```

and now visit http://localhost:5602/ and click on the links above.
