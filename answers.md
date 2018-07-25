## Setup the environment
Having never used Linux or any kind of virtual machine before this exercise, my first goal was to see if I could get the DataDog Agent set up on my Mac operating system. This allowed me to see the GUI and get a clearer understanding of the agent configuration. After removing the OSX agent per the instructions in the [FAQs](https://docs.datadoghq.com/agent/faq/how-do-i-uninstall-the-agent/), I began the installation of a Vagant virtual box and successfully installed the Ubuntu agent with a host name of *precise64*.

## Collecting Metrics:

#### Add Tags to Agent Config File
When I initially tried to open/edit the agent config file using `vi datadog.yaml`, I was denied permission. Recognizing that all other [agent commands for Linux](https://docs.datadoghq.com/agent/basic_agent_usage/ubuntu/) use the `sudo` command, and that its purpose is to access administrative privileges, I was able to access the file using 'sudo vi datadog.yaml'. 

Once accessed, I refrenced a [vi cheat sheet](http://www.lagmonster.org/docs/vi.html) to be able to edit the file, uncomment the tags, and try creating one of my own (region:nw). After restarting the agent and running the info command, I ran into an error:

##### Error: 'did not find expected key'

```
vagrant@precise64:/etc/datadog-agent$ sudo datadog-agent status
Error: unable to set up global agent configuration: unable to load Datadog config file: While parsing config: yaml: line 42: did not find expected key
Usage:
  agent status [flags]
Flags:
  -o, --file string   Output the status command to a file
  -h, --help          help for status
  -j, --json          print out raw json
  -p, --pretty-json   pretty print JSON
Global Flags:
  -c, --cfgpath string   path to directory containing datadog.yaml
  -n, --no-color         disable color output
  ```
  
##### Solution ^
Some Googleing led me to assume that this error had to do with spacing issues in the .yaml file. As it turned out, I had a space (' ') before `tags` and it should have not been indented at all. Reworking the spacing fixed the problem and returned the expected results:

![screen shot 2018-07-18 at 4 50 34 pm](https://user-images.githubusercontent.com/13025907/42913633-daf85c1a-8aaa-11e8-867f-4e54b65586fc.png)

Suggestion for improvement: Having never edited a .yaml file, I was unaware of the spacing sensitivities. A note in the docs re spacing sensitivities (or other common .yaml syntax errors) or instructions on how to enable a linter (if there is one!) that could identify spacing issues would be very helpful.

#### Connecting PostgresSQL

##### Connecting to PostgreSQl from my virtual machine
I quickly realized that typing the 'psql' command from my virtual machine was not providing me the access I required. Using a combination of [this doc](http://suite.opengeo.org/docs/latest/dataadmin/pgGettingStarted/firstconnect.html) and [this doc](https://www.godaddy.com/garage/how-to-install-postgresql-on-ubuntu-14-04/) I was able to create the datadog user. 

##### Database Integration
I chose to integrate the postgreSQL database. I knew that I had postgreSQL installed on my Mac's operating system, but not on the Ubuntu virtual machine. Following this article on [installing psql on Ubuntu](https://www.godaddy.com/garage/how-to-install-postgresql-on-ubuntu-14-04/), I was able to install postgreSQL onto my virtual host.

I quickly ran into a challenge while trying to configure the psql setup:
- The instructions from [PostgresSQL integration docs](https://docs.datadoghq.com/integrations/postgres/) say to `"Edit the postgres.d/conf.yaml file..."` However, there is no `conf.yaml` file to edit; there is only a `postgres.yaml.example` file. 

- I then found a [DataDog blog post (Dec 2017)](https://www.datadoghq.com/blog/collect-postgresql-data-with-datadog/) which advised the following `"Copy the example config file (postgres.yaml.example) and save it as postgres.yaml."` This langauge allowed me to assume that I needed to _copy the example file an re-name it_, although I realized that it refered to the config file as "postgres.yaml" and not "conf.yaml".

- Lastly, there were instructions from the Integration App, which advises to `"Edit conf.d/postgres.yaml"`. 

Thus, there was certainly some confusion about how I should be renaming this file (conf.yaml _vs_ postgres.yaml) and where to store it (postgres.d/ _vs_ conf.d/)!

##### Attempt #1 (conf.d/postres.yaml)
Following the App Integration instructions, I created a *conf.d/postgres.yaml* file. I did not expect this to work, as there were no other .yaml files in this directory yet - only sub-directories. As expected, when restarting the agent and running the info command, there was no change in the status.

##### Attempt #2 (conf.d/postgres.d/conf.yaml):
Next I created a *postgres.d/conf.yaml* file as the Docs advised. After restarting and running the info command to make sure PostgreSQL was configured correctly, postresql was _now_ listed in the Running Checks list. However, I received the following error message:

![image](https://user-images.githubusercontent.com/13025907/42968000-d8141186-8b56-11e8-9867-a5a10765f5e0.png)

##### First try at solving the error ^
Knowing that I was working from a virtual machine, I wanted to ensure that it could connect to PostgreSQL. I found a tip on [this post](https://blog.bigbinary.com/2016/01/23/configure-postgresql-to-allow-remote-connection.html) regarding configuring postgreSQL to allow remote connection. Following this guidance...

1. I located my postgresql config file:

![image](https://user-images.githubusercontent.com/13025907/42968122-36a5b696-8b57-11e8-80bf-a5ab21b16f6b.png)

2. And changed the _listen_address to '*' instead of '5432'
![image](https://user-images.githubusercontent.com/13025907/42968318-c95c8f32-8b57-11e8-8db5-3e4c3c83338e.png)

Unfortunately I still received the same errorr message after restarting the Agent and running the info command.

##### Attempt #3 (postgres.d/postgres.yaml)
Looking back at the conflicting guidance, I decided to try a combination of Doc guidance and the App Integration guidance: I created a *postgres.d/postgres.yaml* file. Finally I received the expected results!

![image](https://user-images.githubusercontent.com/13025907/42976523-cee894c2-8b76-11e8-91b5-beee5f02e036.png)
![screen shot 2018-07-19 at 5 31 40 pm](https://user-images.githubusercontent.com/13025907/42977006-a8493724-8b79-11e8-89a2-cdffe4858719.png)

**Suggestion for Improvement:** While it is understandable that the blog may be outdated (per the nature of most blogs), it would be great if the docs and the integration app both included the guidance re copying the example file _and_ had the correct/same file path and naming convention. This small change would be very helpful!

#### Custom Agent Check
Using this [agent checks documentation](https://docs.datadoghq.com/developers/agent_checks/), I created a check called my_metric, with a my_metric.yaml in the conf.d directory, and a my_metric.py in the checks.d directory. 

![image](https://user-images.githubusercontent.com/13025907/42977145-537c57d4-8b7a-11e8-91e4-250d212b80eb.png)

To ensure that I had created it successfully:

1. I ran the `sudo -u dd-agent -- datadog-agent check <check_name>` command to confirm that my metric was being generated.

![screen shot 2018-07-23 at 7 39 41 pm](https://user-images.githubusercontent.com/13025907/43113768-8dfc5fc4-8eb0-11e8-9a7c-e55d81b6060e.png)

2. I ran the info command to see the status and the checks

![screen shot 2018-07-19 at 11 19 07 am](https://user-images.githubusercontent.com/13025907/42962240-fbd6d3d0-8b45-11e8-9bb8-fa7b4b4354bf.png)

3. I found it in the Metrics Explorer
![screen shot 2018-07-19 at 11 20 27 am](https://user-images.githubusercontent.com/13025907/42962249-00bdafe0-8b46-11e8-9a75-83a2da4036c7.png)

4. I found it in the Metrics Summary
![screen shot 2018-07-19 at 11 23 39 am](https://user-images.githubusercontent.com/13025907/42962363-4bf46594-8b46-11e8-8bff-97848f33c9f3.png)

#### Changing the Collection Interval + Bonus Question

By default, this metric was being submitted every 20 seconds. To change it to every 45 seconds, I edited the my_metric.yaml config file (*not the .py file!*) to include `instances: min_collection_interval: 45`. Looking back at my Metrics Summary (filtered by my_metric), you can see collection happens about every 45 seconds:

![screen shot 2018-07-19 at 5 36 14 pm](https://user-images.githubusercontent.com/13025907/42977153-5b990dfe-8b7a-11e8-982e-3ce67f34e34a.png)
![screen shot 2018-07-19 at 12 20 27 pm](https://user-images.githubusercontent.com/13025907/42965222-38240a62-8b4e-11e8-89ea-e0980f3a8024.png)
![screen shot 2018-07-19 at 12 20 38 pm](https://user-images.githubusercontent.com/13025907/42965223-383f5542-8b4e-11e8-9ec9-4c65dc2347d2.png)

## Visualizing Data:
#### Use Datadog API to create Timeboard
Prior to applying the anamolies and rollup functions to my metrics, I wanted to make sure that I could create a timeboard with the metrics as they were. I saved the results of the api.Timeboard.create() function to a variable `res`, and added `print(res)` at the end of my timeboard.py file. I was glad to see the results in my terminal, which also told me that I could expect a new timeboard in my Dashboard List.

![screen shot 2018-07-24 at 8 05 35 pm](https://user-images.githubusercontent.com/13025907/43177413-ff52a29c-8f7c-11e8-8f38-0a165cea4b12.png)

By clicking and dragging on one of the graphs in My Timeboard, I set the graph to display metrics over the last five minutes. Here is the email that was sent afer taking a snapshot and tagging myself: 

![image](https://user-images.githubusercontent.com/13025907/43176303-d168eb34-8f77-11e8-85a4-1045a9b206b4.png)

After countless tries, unfortunatley I was only able to create a new monitor of my database metric with the anomaly function applied (not a graph on My Timeboard). 


Timeboard:

**[Link to Timeboard](https://app.datadoghq.com/dash/870428/jessis-timeboard?live=true&page=0&is_auto=false&from_ts=1532487269362&to_ts=1532490869362&tile_size=m)

![screen shot 2018-07-24 at 7 46 25 pm](https://user-images.githubusercontent.com/13025907/43176837-60ad4680-8f7a-11e8-8811-ed3191febc83.png)

Metric with Anomaly:

![image](https://user-images.githubusercontent.com/13025907/43176830-529b10c2-8f7a-11e8-81e7-c7706e997c2c.png)


**[CODE: Timeboard and Metric Creation via API](https://gist.github.com/jessiheger/a7cadcf499ad9e81145ef91281f07655)** 

##### Bonus Question: What is the Anomaly graph displaying?
The anomaly function identifies strange behavior in a single metric based on the metric's past performance.

## Monitoring Data
Using the New Monitor option in the Monitors dropdown menu, I was able to create a monitor for My_Metric that sent a warning if the value was over 500 and an alert if over 800. 

![screen shot 2018-07-20 at 4 40 01 pm](https://user-images.githubusercontent.com/13025907/43029493-9cd22c6c-8c3b-11e8-9d9d-c9d375a284cd.png)

#### Bonus Question: Schedule Downtime
By selecting Manage Monitors > my_metric > edit I was able to schedule downtime for the monitoring warnings/alerts.

##### Weekday downtime
![image](https://user-images.githubusercontent.com/13025907/42985760-b08ed528-8ba7-11e8-8cfc-d0560dd597bc.png)

![image](https://user-images.githubusercontent.com/13025907/42985864-148cc54e-8ba8-11e8-816c-ec3f6ed58b19.png)

##### Weekend downtime
![image](https://user-images.githubusercontent.com/13025907/42985748-a8e27582-8ba7-11e8-8103-3f5aadf1bef4.png)

![image](https://user-images.githubusercontent.com/13025907/42985845-fe6523a6-8ba7-11e8-803e-5fa077e3b876.png)

## Collecting APM Data

Following the instructions from the [Flask Installation and Introduction](http://flask.pocoo.org/docs/0.12/quickstart/), I created my virtual environment and ran the python file with python2.7, the default verison on the machine. I continued to get an error message of 'module' object has no attribute 'SSLContext'. After Googling, I found out that by dictating what version of Python the program launched in, I might be able to get around that error. 

By running `ddtrace-run python3 ddflaskapp.py` and installing the various dependencies required, I was able to generate some results in my terminal and a new service in my list.

**Suggestion for Improvement:** It took some time to figure out the workaround to this error, so providing information in the documentation on what version(s) of Python support the Trace Agent would be very helpful! 

### Dashboard with both APM and Infrastructure Metrics

[Link to Timeboard with APM + Infra Metrics](https://app.datadoghq.com/dash/870444/apm-and-infrastructure-dashboard?live=true&page=0&is_auto=false&from_ts=1532487210945&to_ts=1532490810945&tile_size=m)

![screen shot 2018-07-24 at 8 49 30 pm](https://user-images.githubusercontent.com/13025907/43178613-1ef43240-8f83-11e8-8652-2f7ecde11beb.png)

**[CODE: Flask App](https://gist.github.com/jessiheger/decabe25d96a19dd6348a073771f24b5)**

### Bonus Question:
A service is a set of processes that do the same job - for example a web framework or database. A Resource is a particular action for a service.

## Final Question
One potential application that came to mind was the monitoring of natural disaster indicators. Being able to measure temperature and humidity/dryness of the air could help anticipate forrest fires, leading to successful evacuation and preparedness pre-disaster. Measuring precipitation could help ancitipate landslides, and/or measuring the tectonic movement of specific plates/zones over time could help anticipate earthquakes.

A more user-specific application could be tracking the socia media accounts of artists/bands that someone likes or "follows". Many artists rely on social media (Instagram, Twitter, Snapchat) to reach new fans and to notify current ones of exciting updates. But if an user decides they don't want to fill up their newsfeed with band updates, DataDog's anomaly monitoring capabilities could detect an uptick in a user's favorite artists' social media activity, which could imply a new album/song release or an upcoming/ongoing tour!
