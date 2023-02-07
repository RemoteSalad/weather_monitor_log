
## Stage 1:
*   Fill in template configs for base Prometheus and Weather Exporters
    * Weather API key
*   Make first version of docker-compose.yml


## Stage 2:
*   Add Grafana container for another frontend
*   Add Alertmanager
*   Use Secrets for API key

## Stage 3:
*   Create an outflow for Alertmanager to ping slack channel/users (email as well?)
*   Define more complex storage options (updating configs while live, saving of tsdb, dashboard configs)

## Stage 4:
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
