
## &#9745; Stage 1:
*   Fill in template configs for base Prometheus and Weather Exporters
    * Weather API key
*   Make first version of docker-compose.yml


## &#9744; Stage 2:
*   Add Grafana container for another front end
*   Add Alertmanager
*   Use Secrets for API key

## &#9744; Stage 3:
*   Create an outflow for Alertmanager to ping slack channel/users (email as well?)
*   Define more complex storage options (updating configs while live, saving of tsdb, dashboard configs)

## &#9744; Stage 4:
*   Integrate an Logstash/ElasticSearch/Kibana stack for grabbing each container's logs

<br>

---

<br>

## Redundancy:
*   Create redundancy for Prometheus, Grafana, etc.
    * Also the alertmanager?
*   Prometheus redundancy seems like the biggest can of worms. Potential questions are:
    * Do the storage volumes point to the same place or are they separate?
        * does that involve using remote_write/read in addition to docker's named volumes?
    * How to handle metrics duplication?
    * Do we implement a load balancer 'in front' of prom & grafana
    * Related: I believe grafana dodges the redundance issue as it just reads from others, but what of alertmanager? How does that handle the duplication?
    * Terraform, Ansible, Thanos were all mentioned as different tools to handle the variation in configs, deployment, storage duplication, etc
        * Relevant links: <br> 
            https://stackoverflow.com/questions/47215492/how-can-we-get-high-availability-in-prometheus-data-store <br>
            https://medium.com/miro-engineering/prometheus-high-availability-and-fault-tolerance-strategy-long-term-storage-with-victoriametrics-82f6f3f0409e
            https://hevodata.com/learn/prometheus-high-availability/ <br>
            https://prometheus.io/docs/prometheus/latest/federation/


## Unscheduled Tasks:
*   Dockerfile/.dockerignore unneeded? I don't believe I am building any custom images
    *   Create a custom exporter for https://open-meteo.com/ instead of using the pre-exiting exporter for openweathermap.org
    *   Modify the openweather exporter to handle live config updates like nodemon (pymon if python):
        *   https://pypi.org/project/py-mon/
        *   https://stackoverflow.com/questions/49355010/how-do-i-watch-python-source-code-files-and-restart-when-i-save
        *   https://stackoverflow.com/questions/33584540/tools-similar-to-nodemon-for-perl
        *   Potentially rebuild so the runtime flags in compose aren't quite as verbose
*   Create some unified front end
    * i.e. resources are accessed through a frontend container instead of having the ip & port to the prometheus container, grafana container, etc.
    * use a Nginx exporter to observe site traffic
*   Look & Understand the OpenWeatherMap exporter code
    *   Compare the output of the exporter and usual api call
*   Document the creation of volumes externally as they cannot be included in the compose file; mountpoints are in compsoe
    *   $docker volume create grafana-weather
        $docker volume create prometheus-weather-tsdb
*   Rename prom & graf containers to not be so generic -> imagine having other projects with those container hanging out?

*   CLI for the grafana docker independent: <tt> docker run -dp 3000:3000 --name=grafana --network weather_monitor_log_default -v grafana-weather:/var/lib/grafana grafana/grafana-oss </tt>

*   Add custom grafana plug-ins to the prebuild? Somehow have those relevant plugins to the dashboard?  Better would be to have that built in somehow with a lot of predefined dashboards (i.e. the end user would have minor set up,[define target cities, put in api key, create volumes/bind mounts --- Hey that's an idea. put all this into a little script that runs at the beginning] and then start with a more or less set up dashboard) - but still something not so clunky as to prevent using more recent grafana/prometheus images.

*   Review naming schemas for consistent usages (e.g. prometheus volume is named with an underscore while grafana's is a hyphenated)
*   Create Schema map using Markdown's mermaid

## Goals:
*   Use Docker Secrets
*   Mix use of bind mounts and named volumes
    * How are Grafana dashboards stored?
    * updating configs and saving stateful data/longterm storage interests
    * implementing a pretend "offsite" storage target
    * Related to NGINX scraper?
*   Explore how Docker containers work via network
*   Static implementation or is dynamic viable?
*   Explore exporter SDK
*   Practice more complex PromQL expressions, Grafana visualizations
*   Find some place to implement other exporters, stateful storage

A question: How do all of these manage bad configs getting pushed to them while they are live? Do they break or continue on with the previously working config? I want a pretty sturdy understanding

Multi-target exporter? ElasticSearch Exporter?

Is Apple's Weatherkit also an option? (https://developer.apple.com/weatherkit/)

Add versions + dependancies list

Trying to use swarm with secrets failed. Ugly store the apikey for now then bind mount later

Try simulating the exporters as offsite (containers across two 2 networks), then port forward/reverse tunnel to have them appear available as on site for the scraping/visualization side

<br>

---

<br>

## Reports:

### TSDB Confusion After Adding "command: --web.enable-lifecycle" to Prometheus Service:

<blockquote>

&emsp;&emsp;&emsp;<ins>2023/2/9</ins>: Because of my unsteadiness surrounding the persistance of Docker's named volumes and the desire to utilize Prometheus's built-in ability to load and use updated config files (including keeping a cached config to use in case the new config is faulty), I tried to implement it in the <tt>service.command</tt> field of the <tt>docker-compose.yml</tt> yesterday. Once I confirmed the syntax for that field and restarted the service using <tt>docker compose up -d</tt> I encountered an issue. Querying the Prometheus service I could not find the past few days of metrics. This was very concerning because I do not yet trust Docker's storage management. Further, I had also changed the bind mount target of the config file for the service, as I wanted to start incorporating alerting and recording rules. (bind mounting a file overwrote the default file in the container, I was concerned doing so for a directory similarly overwrote the directory's contents. I need to follow up regarding mounting directories with and without trailing forward slashes)<br>
&emsp;&emsp;&emsp;This led me to panic a bit, and after reverting the docker-compose changes and confirming it was working. I stopped working on the project and tried to articulate the potential interactions that caused this issue, as well as a few other questions that could not be pursued given this more pressing issue. I saved the terminal logs to refer back to the output from the container. While referring to the documentation I created prometheus containers while incrementally changing parameters until I discovered that adding command flags overrides all of the 'default' ones designated in the image's Dockerfile. [This](https://docs.docker.com/compose/compose-file/#command) is the part of the documentation that clearly states that behavior.<br>
&emsp;&emsp;&emsp;Further I observed some of Prometheus's tsdb storage behavior. Primarily how Prometheus makes 2 hour chunks of data (in those uniquely named directories), uses the wal file (write-ahead log) for saving data before reaching the two hour mark, and then later compacts those directories into more effecient packages later on. It also seems to write out wal to a storage directory before the two hour time when asked to shutdown. I found that the promtool can be used to back-fill designated recording rules, as they only begin to be generated after a rule is set. I encountered a few potential solutions to running redundant (federated) prometheus with a shared final storage. I have a few supplementary exercises I want to try now, integrating those mis-located metrics data; making a script and cron job to create, check and save each of the docker containers (e.g. use cut and diff to efficiently compare logs between the containers, show the internal file structure, get the runtime sizes of the containers), and how modifying volumes propagates between containers. Also there seems to be some nuance between the different places commanads are passed to a container, "shell form" vs "exec form", and I need to understand that better.

</blockquote>

<!-- Some scratch that went into the report. Hiding for clarity on preview

<ins>abc</ins> how to underline
> to block quote
--try git for that testing directory, esp to make a script + cron job to run and test the different containers -> output their logs

saving logs; storage; active queries, reading
testing methodology, steps, storage, tsdb behavior(head, compaction,), 
exec form vs shell form for Dockerfile CMD, docerk exec, compose.service.command, docker run commands

There are some trailing questions I would like to resolve. This is solved by giving the rule configs' file paths absolutely, but I am still curious how relative paths are resolved. Is it relative to the config file's location, or the path where Prometheus was executed? (surely the latter - it is the working directory after all)

why is the development named volume /prometheus/prometheus.yml executable? a result of having web.enable-lifecycle?

volume share modification in a running container, exec vs shell form, 405 when trying to curl a running container

remote write/read and some other goals  for having a more interesting virtualized project. federation/HA prometheus
-->

<br>

---

<br>

**URLs**:
*   Weather:
    *   https://openweathermap.org/api
    *   https://open-meteo.com/
    *   https://github.com/RichiH/openweathermap_exporter
    *   https://hub.docker.com/r/billykwooten/openweather-exporter
    *   https://github.com/billykwooten/GrafanaDashboards/blob/master/open_weather_map.json
    *    https://github.com/briandowns/openweathermap
*   Prometheus:
    *   https://hub.docker.com/r/prom/prometheus
    *   https://hub.docker.com/u/prom
    *   https://prometheus.io/docs/instrumenting/exporters/
    *   https://github.com/prometheus/alertmanager
    *   https://github.com/prometheus/prometheus
    *   https://prometheus.io/docs/instrumenting/writing_exporters/
    *   https://www.opsramp.com/guides/prometheus-monitoring/prometheus-blackbox-exporter/
*   Grafana:
    *   https://hub.docker.com/r/grafana/grafana
*   Nginx Exporter:
    *   https://github.com/nginxinc/nginx-prometheus-exporter
*   Docker:
    *   https://docs.docker.com/engine/swarm/secrets/
    *   https://earthly.dev/blog/docker-secrets/
    *   https://docs.docker.com/storage/volumes/
    *   https://docs.docker.com/storage/bind-mounts/
    *   https://docs.docker.com/config/daemon/prometheus/
*   ElasticSearch:
    *   https://hub.docker.com/_/elasticsearch
    *   https://github.com/prometheus-community/elasticsearch_exporter

## Check-in as of 2/21 & 2/22

<blockquote>

&emsp;&emsp;&emsp; Much of the past week has been spent on understanding the Elastic stack. Right now I am writing a debrief as I now have two filebeat containers and a logstash container integrated into this compose project (the filebeats need to be redundant as they cannot have duplicate outputs - perhaps this can be made more efficient by logstash forwarding both unprocessed logs and processed logs?). As things are now working and generating something of my own to practice delving in to using ES & Kibana, I want to take the chance to sort my thoughts.<br>
&emsp;&emsp;&emsp; While the documentation seemed inviting and implied a certain ease, it quickly became clear that there is a growing multitude of implementations for this logging/observation ecosystem, with a widening field of knowledge(one of their newest recommended applications was the Elastic-agent; a heavy compilation of their metrics&logging acquisition Beats & Logstash). More importantly there is a vast library of old or bad documentation, outdated quickstart/tutorials that always suggest trying their hosted service.
&emsp;&emsp;&emsp; Some quick things to note before moving on:

but for tomorrow
running locally/bare on vm; kibana container ok; elastic container too heavy; , creating a new vm for ES then file beat that could connect to ES cloud (had to be through cloud_id and auth); then porting it all back and cleaning it up and testing it on the og vm
-feature of logstash before containerization -> run on host and forward updates to the logs to ES (part of why conceptualizing was hard and the implementation felt clunky.)

cleaning up git project repo. 
Using recommended logstash output index=> ... now conflicts with datastream/elastic manages the indexing

</blockquote>

I had to create a keystore in an instance of the logstash container with the appropriate credentials, and then copy the entire contents of the /usr/share/logstash/config directory into the host project. Now I bind mount that directory on the host (including the default files I haven't changed) because directly binding the logstash.keystore was not recommonded. That encrypted file with the needed credentials

Why did I make this change to using the logstash keystore? Passing the cloud credentials & authentication as referenced environment variables worked except that the Logstash application replaced some of those referenced values to literals during runtime - i.e. visible in the projects Github. That said, I could not bind mount the config as read only. This solution allows contianers to be reproduced the same as before(from a single compose command), with cloud integration, without having secrets revealed inadvertantly.