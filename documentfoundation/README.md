# Deployment of Grimoire Open Development Analytics platform for The Document Foundation

## 1. Introduction

This brief document explains how to deploy the Grimoire Open Development Analytics platform for [The Document Foundation](https://www.documentfoundation.org/). These are the main steps included in the following sections:
* Installation of the docker container
* Data retrieval
* Data enrichment
* Publication on a Kibana dashboard customized by Bitergia

## 2. Requirements

* The host machine should be a GNU/Linux box with at least: nginx, docker(>=1.5) and docker-compose (>= 1.5)
* You must have a gerrit account in gerrit.libreoffice.org with your ssh public key added to it.
* 50 GB hard disk free space for git repositories.
* Chrome/Chromium is the web browser recommended for performance.

## 3. Install the docker containers

Before deploying the container in charge of running the Bitergia stack you'll need the ones below:
```
root@vm167:/docker/containers# cat docker-compose.yml |grep image
  image: mariadb:10.0
  image: elasticsearch:2.2
  image: bitergia/kibiter:4.4.1-public
  image: bitergia/kibiter:4.4.1
```

The Mordred container links two of the listed above and needs some data persistency. We want the host to share:
* the configuration directory
* the logs directory
* the cache used by the data collector AKA Perceval
* the SSH keys needed to collect data from the Gerrit server (we'll need a public key already set up to access gerrit because it does use ssh as authentication)

```
root@vm167:/docker/mordred# cat docker-compose.yml 
mordred:
  #restart: "always"
  image: bitergia/mordred:latest
  volumes:
    - /docker/mordred:/home/bitergia/conf
    - /docker/logs:/home/bitergia/logs
    - /docker/storage/tdf/perceval-cache:/home/bitergia/.perceval
    - /docker/storage/tdf/perceval-ssh:/home/bitergia/.ssh
  external_links:
    - containers_mariadb_1:mariadb
    - containers_elasticsearch_1:elasticsearch
```

## 4. Set up the dashboard generator

Mordred (one of the bad guys in the Arthurian legend) is in charge of the configuration and execution of the different components. Have a look at its documentation because we'll get to the point in this section (https://github.com/Bitergia/mordred/blob/master/docker/README.md )

### 4.1 setup.cfg

This is the file you need to start storing data into a local elasticsearch for the data sources Git, Gerrit and Bugzilla. The project file is named 'projects.json' and is one of the files "mounted" using the docker volume.

```
[general]
short_name = tdf
update = true
min_update_delay = 600
debug = true
logs_dir = /home/bitergia/logs
# yyyy-mm-dd
from_date = 

[projects]
projects_file = /home/bitergia/conf/projects.json

[es_collection]
url = http://elasticsearch:9200

[es_enrichment]                                                                                                          
url = http://elasticsearch:9200                                                                                          
autorefresh = false                                                                                                      
studies = true                                                                                                           
                                                                                                                         
[sortinghat]                                                                                                             
host = mariadb                                                                                                           
user = root                                                                                                              
password =                                                                                                               
database = tdf_sh                                                                                                        
load_orgs = true                                                                                                         
orgs_file = /home/bitergia/conf/orgs_file                                                                                
identities_file = /home/bitergia/conf/sh-gitdm-libreoffice.json                                                          
autoprofile = gitdm:libreoffice, git                                                                                     
matching = email-name                                                                                                    
sleep_for = 86400                                                                                                        
bots_names =                                                                                                             
                                                                                                                         
[phases]                                                                                                                 
collection = true                                                                                                        
identities = true                                                                                                        
enrichment = true                                                                                                        
panels = true

[git]                                                                                                                    
raw_index = git_170123                                                                                                   
enriched_index = git_170123_enriched_170123                                                                              
                                                                                                                         
[bugzilla]
raw_index = bugzilla_new
enriched_index = bugzilla_new_enriched_161130
backend-user = owlbot@bitergia.com
backend-password = *********

[gerrit]
raw_index = gerrit_170123
enriched_index = gerrit_170123_enriched_170123
user = tdfbot
```
### 4.2 requirements.cfg

The latest version available at the moment of writting this is named 'catwoman.beta'. See Mordred's documentation for more info about releases and upgrades.

```
root@vm167:/docker/mordred# cat requirements.cfg 
#!/bin/bash                                                                                                              
                                                                                                                         
RELEASE='catwoman.beta' 
```

### 4.3 projects.json

The project definition and the name of the different data sources as are understood by Perceval (https://github.com/grimoirelab/perceval#usage)
```
{
    "TheDocumentFoundation": {
        "bugzilla": [
            "https://bugs.documentfoundation.org/"
        ], 
        "git": [
            "git://gerrit.libreoffice.org/All-Users", 
            "git://gerrit.libreoffice.org/benchmark", 
            "git://gerrit.libreoffice.org/bibisect-macosx-64-5.0", 
            "git://gerrit.libreoffice.org/bibisect-macosx-64-5.1", 
            "git://gerrit.libreoffice.org/bibisect-macosx-64-5.2", 
            "git://gerrit.libreoffice.org/bibisect-win32-5.0", 
            "git://gerrit.libreoffice.org/bibisect-win32-5.1", 
            "git://gerrit.libreoffice.org/bibisect-win32-5.2", 
            "git://gerrit.libreoffice.org/binfilter", 
            "git://gerrit.libreoffice.org/bugzilla", 
            "git://gerrit.libreoffice.org/buildbot", 
            "git://gerrit.libreoffice.org/core", 
            "git://gerrit.libreoffice.org/cppcheck-reports", 
            "git://gerrit.libreoffice.org/cppunit", 
            "git://gerrit.libreoffice.org/dcount", 
            "git://gerrit.libreoffice.org/dev-tools", 
            "git://gerrit.libreoffice.org/devcentral", 
            "git://gerrit.libreoffice.org/dictionaries", 
            "git://gerrit.libreoffice.org/gerrit-etc", 
            "git://gerrit.libreoffice.org/gnu-make-lo", 
            "git://gerrit.libreoffice.org/help", 
            "git://gerrit.libreoffice.org/impress_remote", 
            "git://gerrit.libreoffice.org/libabw", 
            "git://gerrit.libreoffice.org/libabw-test", 
            "git://gerrit.libreoffice.org/libcdr", 
            "git://gerrit.libreoffice.org/libcdr-test", 
            "git://gerrit.libreoffice.org/libetonyek", 
            "git://gerrit.libreoffice.org/libetonyek-test", 
            "git://gerrit.libreoffice.org/libexttextcat", 
            "git://gerrit.libreoffice.org/libfreehand", 
            "git://gerrit.libreoffice.org/libfreehand-test", 
            "git://gerrit.libreoffice.org/libgltf", 
            "git://gerrit.libreoffice.org/libmspub", 
            "git://gerrit.libreoffice.org/libmspub-test", 
            "git://gerrit.libreoffice.org/libpagemaker", 
            "git://gerrit.libreoffice.org/libpagemaker-test", 
            "git://gerrit.libreoffice.org/libvisio", 
            "git://gerrit.libreoffice.org/libvisio-test", 
            "git://gerrit.libreoffice.org/libzmf", 
            "git://gerrit.libreoffice.org/libzmf-test", 
            "git://gerrit.libreoffice.org/lode", 
            "git://gerrit.libreoffice.org/mcm-database", 
            "git://gerrit.libreoffice.org/mso-dumper", 
            "git://gerrit.libreoffice.org/officeotron", 
            "git://gerrit.libreoffice.org/online", 
            "git://gerrit.libreoffice.org/original-artwork", 
            "git://gerrit.libreoffice.org/perfdbmgr", 
            "git://gerrit.libreoffice.org/sdk-examples", 
            "git://gerrit.libreoffice.org/si-gui", 
            "git://gerrit.libreoffice.org/tb3-django", 
            "git://gerrit.libreoffice.org/tb3-docker", 
            "git://gerrit.libreoffice.org/templates", 
            "git://gerrit.libreoffice.org/test-files", 
            "git://gerrit.libreoffice.org/translations", 
            "git://gerrit.libreoffice.org/ui-test", 
            "git://gerrit.libreoffice.org/voting"
        ],
        "gerrit": [
            "gerrit.libreoffice.org"
        ],
        "meta": {
            "title": "TheDocumentFoundation"
        }
    }
}
```

### 4.4 Sorting hat files

In the setup.cfg file above we define two inputs for the tool in charge of unifying the identities and affilation (its name is Sortinghat https://github.com/MetricsGrimoire/sortinghat). 

Those are the files (both available in the container's directory /home/bitergia/conf) :
* orgs_file: with a mapping between domains and company/organization names
* sh-gitdm-libreoffice.json: with the information offered by gitdm exported to a sortinghat file

## 5. Execute it

Start the docker containers and the result should be available while you grab a tea or coffee.

```
root@vm167:/docker/mordred# docker-compose up -d
Creating mordred_mordred_1
```

The log /tmp/mordred.log will say something like ..
```
root@vm167:/docker/mordred# tail -n4 /docker/logs/mordred.log 
2017-01-24 19:20:21,563 - mordred - INFO - [gerrit] Gathering identities from raw data                                   
2017-01-24 19:20:21,774 - mordred - INFO - [gerrit] enrichment starts                                                    
2017-01-24 19:20:21,775 - mordred - DEBUG - [gerrit] enrichment starts for gerrit.libreoffice.org                        
2017-01-24 19:20:28,157 - mordred - INFO - [gerrit] enrichment finished in 00:00:06
```

## 6. Make it available to your friends!

As soon as you ElasticSearch starts to have data, you'll need to make Kibiter available to the world. You can use a simple nginx configuration like the one below

```
server {                                                                                                                                                                                          
        listen 443;                                                                                                                                                                          
        server_name vm167.documentfoundation.org;                                                                                                                                                 
                                                                                                                                                                                                  
        ssl    on;                                                                                                                                                                                
        ssl_certificate    /etc/nginx/ssl/nginx.crt;                                                                                                                                              
        ssl_certificate_key    /etc/nginx/ssl/nginx.key;                                                                                                                                          
                                                                                                                                                                                                  
        rewrite ^/$ /app/kibana#/dashboard/Overview permanent;                                                                                                                                    
                                                                                                                                                                                                  
        location / {                                                                                                                                                                              
                proxy_pass http://127.0.0.1:5601/;                                                                                                                                                
                proxy_redirect http://127.0.0.1:5601/ /;                                                                                                                                          
                proxy_http_version 1.1;                                                                                                                                                           
                proxy_set_header Upgrade $http_upgrade;                                                                                                                                           
                proxy_set_header Connection 'upgrade';                                                                                                                                            
                proxy_set_header Host $host;                                                                                                                                                      
                proxy_cache_bypass $http_upgrade;                                                                                                                                                 
        }                                                                                                                                                                                         
                                                                                                                                                                                                  
        location /edit/ {                                                                                                                                                                         
                auth_basic "Edition mode - Bitergia dashboard for TDF";                                                                                                                           
                auth_basic_user_file /home/gelk/docker/containers/dashboard-password;                                                                                                             
                        proxy_pass http://127.0.0.1:5602/;                                                                                                                                        
                proxy_redirect http://127.0.0.1:5602/ /edit/;                                                                                                                                     
                proxy_http_version 1.1;                                                                                                                                                           
                proxy_set_header Upgrade $http_upgrade;                                                                                                                                           
                proxy_set_header Connection 'upgrade';                                                                                                                                            
                proxy_set_header Host $host;                                                                                      
                proxy_cache_bypass $http_upgrade;                                                                                                                                                 
        }  
}
```
