# HW: Search Engine

In this assignment you will create a highly scalable web search engine.

**Due Date:** Sunday, 9 May

**Learning Objectives:**
1. Learn to work with a moderate large software project
1. Learn to parallelize data analysis work off the database
1. Learn to work with WARC files and the multi-petabyte common crawl dataset
1. Increase familiarity with indexes and rollup tables for speeding up queries

## Task 0: project setup

1. Fork this github repo, and clone your fork onto the lambda server

1. Ensure that you'll have enough free disk space by:
    1. bring down any running docker containers
    1. run the command
       ```
       $ docker system prune
       ```

## Task 1: getting the system running

In this first task, you will bring up all the docker containers and verify that everything works.

There are three docker-compose files in this repo:
1. `docker-compose.yml` defines the database and pg_bouncer services
1. `docker-compose.override.yml` defines the development flask web app
1. `docker-compose.prod.yml` defines the production flask web app served by nginx

Your tasks are to:

1. Modify the `docker-compose.override.yml` file so that the port exposed by the flask service is different.

1. Run the script `scripts/create_passwords.sh` to generate a new production password for the database.

1. Build and bring up the docker containers.

1. Enable ssh port forwarding so that your local computer can connect to the running flask app.

1. Use firefox on your local computer to connect to the running flask webpage.
   If you've done the previous steps correctly,
   all the buttons on the webpage should work without giving you any error messages,
   but there won't be any data displayed when you search.

1. Run the script
   ```
   $ sh scripts/check_web_endpoints.sh
   ```
   to perform automated checks that the system is running correctly.
   All tests should report `[pass]`.

## Task 2: loading data

There are two services for loading data:
1. `downloader_warc` loads an entire WARC file into the database; typically, this will be about 100,000 urls from many different hosts. 
1. `downloader_host` searches the all WARC entries in either the common crawl or internet archive that match a particular pattern, and adds all of them into the database

### Task 2a

We'll start with the `downloader_warc` service.
There are two important files in this service:
1. `services/downloader_warc/downloader_warc.py` contains the python code that actually does the insertion
1. `downloader_warc.sh` is a bash script that starts up a new docker container connected to the database, then runs the `downloader_warc.py` file inside that container

Next follow these steps:
1. Visit https://commoncrawl.org/the-data/get-started/
1. Find the url of a WARC file.
   On the common crawl website, the paths to WARC files are referenced from the Amazon S3 bucket.
   In order to get a valid HTTP url, you'll need to prepend `https://commoncrawl.s3.amazonaws.com/` to the front of the path.
1. Then, run the command
   ```
   $ ./download_warc.sh $URL
   ```
   where `$URL` is the url to your selected WARC file.
1. Run the command
   ```
   $ docker ps
   ```
   to verify that the docker container is running.
1. Repeat these steps to download at least 5 different WARC files, each from different years.
   Each of these downloads will spawn its own docker container and can happen in parallel.

You can verify that your system is working with the following tasks.
(Note that they are listed in order of how soon you will start seeing results for them.)
1. Running `docker logs` on your `download_warc` containers.
1. Run the query
   ```
   SELECT count(*) FROM metahtml;
   ```
   in psql.
1. Visit your webpage in firefox and verify that search terms are now getting returned.

### Task 2b

The `download_warc` service above downloads many urls quickly, but they are mostly low-quality urls.
For example, most URLs do not include the date they were published, and so their contents will not be reflected in the ngrams graph.
In this task, you will implement and run the `download_host` service for downloading high quality urls.

1. The file `services/downloader_host/downloader_host.py` has 3 `FIXME` statements.
   You will have to complete the code in these statements to make the python script correctly insert WARC records into the database.

   HINT:
   The code will require that you use functions from the cdx_toolkit library.
   You can find the documentation [here](https://pypi.org/project/cdx-toolkit/).
   You can also reference the `download_warc` service for hints,
   since this service accomplishes a similar task.

1. Run the query
   ```
   SELECT * FROM metahtml_test_summary_host;
   ```
   to display all of the hosts for which the metahtml library has test cases proving it is able to extract publication dates.
   Note that the command above lists the hosts in key syntax form, and you'll have to convert the host into standard form.
1. Select 5 hostnames from the list above, then run the command
   ```
   $ ./downloader_host.sh "$HOST/*"
   ```
   to insert the urls from these 5 hostnames.

## ~~Task 3: speeding up the webpage~~

Since everyone seems pretty overworked right now,
I've done this step for you.

There are two steps:
1. create indexes for the fast text search
1. create materialized views for the `count(*)` queries

## Submission

1. Edit this README file with the results of the following queries in psql.
   The results of these queries will be used to determine if you've completed the previous steps correctly.

    1. This query shows the total number of webpages loaded:
       ```
       select count(*) from metahtml;
       ```
       ``` 
       novichenko=# select count(*) from metahtml;
         count
       ---------
        1636699
       (1 row)
        ```

    1. This query shows the number of webpages loaded / hour:
       ```
       select * from metahtml_rollup_insert order by insert_hour desc limit 100;
       ```
       ```
        hll_count |  url  | hostpathquery | hostpath | host  |      insert_hour
       -----------+-------+---------------+----------+-------+------------------------
                4 |  1266 |          1314 |     1307 |     4 | 2021-05-06 16:00:00+00
                4 | 20774 |         20700 |    19322 |     4 | 2021-05-06 15:00:00+00
                4 | 24205 |         23659 |    21097 |     4 | 2021-05-06 14:00:00+00
                4 | 22787 |         24478 |    22854 |     4 | 2021-05-06 13:00:00+00
                4 | 22961 |         23470 |    22458 |     4 | 2021-05-06 12:00:00+00
                4 | 22444 |         23164 |    22742 |     4 | 2021-05-06 11:00:00+00
                4 | 23430 |         23307 |    19534 |     4 | 2021-05-06 10:00:00+00
                4 | 22561 |         22273 |    21349 |     4 | 2021-05-06 09:00:00+00
                4 | 22192 |         21598 |    20811 |     4 | 2021-05-06 08:00:00+00
                4 | 21147 |         21606 |    20364 |     4 | 2021-05-06 07:00:00+00
                4 | 21482 |         21477 |    20802 |     4 | 2021-05-06 06:00:00+00
                4 | 22264 |         22447 |    17117 |     4 | 2021-05-06 05:00:00+00
                4 | 21374 |         22275 |    21005 |     4 | 2021-05-06 04:00:00+00
                4 | 22109 |         21750 |    20909 |     4 | 2021-05-06 03:00:00+00
                4 | 21888 |         21831 |    20980 |     4 | 2021-05-06 02:00:00+00
                4 | 22032 |         21594 |    18993 |     4 | 2021-05-06 01:00:00+00
                4 | 21464 |         22346 |    19362 |     4 | 2021-05-06 00:00:00+00
                4 | 22950 |         21787 |    20743 |     4 | 2021-05-05 23:00:00+00
                5 | 25109 |         25883 |    25232 |     5 | 2021-05-05 22:00:00+00
                5 | 25541 |         24438 |    23883 |     5 | 2021-05-05 21:00:00+00
                6 | 30756 |         30605 |    26997 |     6 | 2021-05-05 20:00:00+00
                6 | 31284 |         30927 |    30720 |     6 | 2021-05-05 19:00:00+00
                6 | 30544 |         32555 |    31570 |     6 | 2021-05-05 18:00:00+00
                6 | 31385 |         31721 |    27668 |     6 | 2021-05-05 17:00:00+00
                8 | 33737 |         33277 |    32102 |     8 | 2021-05-05 16:00:00+00
                8 | 39832 |         40009 |    36841 |     9 | 2021-05-05 15:00:00+00
                9 | 45958 |         44502 |    43339 |     9 | 2021-05-05 14:00:00+00
                9 | 38089 |         35821 |    35152 |     9 | 2021-05-05 13:00:00+00
                9 | 52435 |         52122 |    48025 |     9 | 2021-05-05 12:00:00+00
                9 | 50761 |         50830 |    50401 |     9 | 2021-05-05 11:00:00+00
                9 | 53050 |         51462 |    47437 |     9 | 2021-05-05 10:00:00+00
                9 | 53509 |         52913 |    47329 |     9 | 2021-05-05 09:00:00+00
                9 | 51582 |         51472 |    49212 |     9 | 2021-05-05 08:00:00+00
                9 | 54059 |         51530 |    50059 |     9 | 2021-05-05 07:00:00+00
                9 | 52236 |         50722 |    49848 |     9 | 2021-05-05 06:00:00+00
                9 | 41990 |         40941 |    35991 |     9 | 2021-05-05 05:00:00+00
                4 | 26356 |         28119 |    23275 |     4 | 2021-05-05 04:00:00+00
                4 | 22184 |         22621 |    19987 |     4 | 2021-05-05 03:00:00+00
                3 |  5708 |          5739 |     5543 |     3 | 2021-05-05 02:00:00+00
                1 | 54751 |         53205 |    51758 | 47187 | 2021-05-04 21:00:00+00
                1 | 12285 |         12735 |    12348 | 10759 | 2021-05-04 16:00:00+00
                1 | 45895 |         46070 |    45774 | 39793 | 2021-05-04 15:00:00+00
                1 | 23059 |         23725 |    22713 | 19287 | 2021-05-04 06:00:00+00
                2 | 67191 |         66047 |    63674 | 58507 | 2021-05-04 04:00:00+00
                2 | 65836 |         65917 |    64344 | 56221 | 2021-05-04 03:00:00+00
                3 | 83589 |         85706 |    82145 | 70355 | 2021-05-03 23:00:00+00
                1 |  2019 |          2052 |     1982 |  1760 | 2021-05-03 22:00:00+00
       (47 rows) 
       ```


    1. This query shows the hostnames that you have downloaded the most webpages from:
       ```
       select * from metahtml_rollup_host order by hostpath desc limit 100;
       
       ```
       ```
         url   | hostpathquery | hostpath |                 host
       --------+---------------+----------+--------------------------------------
        216240 |        221394 |   203057 | com,nytimes)
        180735 |        179526 |   152375 | com,forbes)
        103407 |        104098 |   103434 | com,newstimes)
         73724 |         75439 |    74531 | com,latimes)
        104235 |        103822 |    70879 | com,vanityfair)
         62939 |         64386 |    60962 | com,bt)
         62335 |         57941 |    57357 | com,huffpost)
         30643 |         30471 |    30471 | com,moviebreakingnews)
         21132 |         20848 |    19819 | com,gameslist)
          1023 |          1031 |     1021 | es,abc)
            96 |            96 |       96 | jp,co,infoseek,news)
            60 |            60 |       60 | ru,yandex,travel)
            57 |            57 |       57 | com,pinterest,pl)
            53 |            53 |       53 | com,catholic,forums)
            53 |            53 |       53 | jp,okwave,sp)
            48 |            48 |       48 | uk,ac,icr,cansarblack)
            45 |            45 |       45 | com,vmware,docs)
            43 |            43 |       43 | au,com,fishpond)
            39 |            39 |       39 | edu,asu,aztransmac2)
            38 |            38 |       38 | gr,news247)
            36 |            36 |       36 | com,lavanguardia,hemeroteca)
            41 |            41 |       35 | com,herokuapp,worldpackersplatform)
            35 |            35 |       35 | com,verizon,tv)
            34 |            34 |       34 | com,shopify,community)
            33 |            33 |       33 | com,petfinder,pro)
            33 |            33 |       33 | com,aliexpress)
            32 |            32 |       32 | es,laopinioncoruna)
            31 |            31 |       31 | com,tableau,community)
            30 |            30 |       30 | com,xrea,s1009,suneo9)
            30 |            30 |       30 | com,xrea,s601,btmon)
            29 |            29 |       29 | be,onroerenderfgoed,inventaris)
            27 |            27 |       27 | com,themrphone)
            27 |            27 |       27 | com,motherjones,fullsite)
            26 |            26 |       26 | edu,ttu,scholars)
            26 |            26 |       26 | com,its-mo)
            25 |            25 |       25 | com,infogram)
            25 |            25 |       25 | com,motherjones,develop)
            25 |            25 |       25 | mil,dtic,apps)
            25 |            25 |       25 | it,unict,iris)
            25 |            25 |       25 | org,sportsvideo,dev)
            25 |            25 |       25 | live,588490)
            24 |            24 |       24 | com,fiverr,forum)
            24 |            24 |       24 | com,exxonmobil)
            24 |            24 |       24 | ca,uwo,politicalscience)
            24 |            24 |       24 | mx,com,eleconomista)
            23 |            23 |       23 | edu,harvard,news)
            23 |            23 |       23 | link,app,dote)
            23 |            23 |       23 | br,com,clicrbs,gauchazh)
            23 |            23 |       23 | com,washingtoncitypaper)
            23 |            23 |       23 | com,adobe,community)
            22 |            22 |       22 | gov,nih,nlm,ncbi,pubmed)
            22 |            22 |       22 | ua,meta,testlib)
            22 |            22 |       22 | com,woot,forums)
            22 |            22 |       22 | com,stylebistro)
            21 |            21 |       21 | com,sap,blogs)
            21 |            21 |       21 | to,land,if,botubox)
            23 |            23 |       21 | com,blogspot,entbiz)
            21 |            21 |       21 | org,datacite,search)
            20 |            20 |       20 | com,nettiauto)
            20 |            20 |       20 | com,ritetag)
            20 |            20 |       20 | kg,cars)
            20 |            20 |       20 | ru,booknigi)
            20 |            20 |       20 | com,makezine)
            21 |            21 |       19 | com,blogspot,jesusnosso)
            19 |            19 |       19 | com,blogspot,bleakbliss)
            19 |            19 |       19 | pt,folhadodomingo)
            19 |            19 |       19 | com,blogspot,gonzaloraffoinfonews)
            19 |            19 |       19 | com,rapidtables)
            19 |            19 |       19 | in,learncbse,ask)
            18 |            18 |       18 | edu,msu,usd)
            18 |            18 |       18 | de,globetrotter)
            18 |            18 |       18 | com,webempresa)
            18 |            18 |       18 | ru,invitro)
            18 |            18 |       18 | com,fjsen,auto)
            18 |            18 |       18 | com,homebrewfinds)
            18 |            18 |       18 | com,baomoi)
            18 |            18 |       18 | org,gentoo,au,rsync1)
            18 |            18 |       18 | br,com,clubedeautores)
            18 |            18 |       18 | life,medley)
            18 |            18 |       18 | net,sympaty,love)
            17 |            17 |       17 | org,ada,findadentist)
            17 |            17 |       17 | hr,nsk,katalog)
            17 |            17 |       17 | com,cirkwi)
            17 |            17 |       17 | com,nbcbayarea)
            17 |            17 |       17 | com,manyland)
            17 |            17 |       17 | cn,com,jhysw)
            16 |            16 |       16 | com,elpais)
            16 |            16 |       16 | com,threadless,fashionedbynature)
            16 |            16 |       16 | com,wwwhatsnew)
            16 |            16 |       16 | com,summitdaily)
            16 |            16 |       16 | lt,technologijos)
            16 |            16 |       16 | com,made-in-china,sa)
            16 |            16 |       16 | com,anotherdotcom,maggiesfarm)
            16 |            16 |       16 | hu,mta,fenyeve)
            16 |            16 |       16 | com,tasteofhome,45air-www)
            16 |            16 |       16 | de,spielplatztreff)
            16 |            16 |       16 | ru,6030000)
            16 |            16 |       16 | com,thefreedictionary,acronyms)
            16 |            16 |       16 | com,babycenter)
            16 |            16 |       16 | au,edu,griffith,research-repository)
      (100 rows)
      ```



1. Take a screenshot of an interesting search result.
   Add the screenshot to your git repo, and modify the `<img>` tag below to point to the screenshot.

   <img src='Comics_Query.PNG' />
   <img src='Comics_results.PNG' />

1. Commit and push your changes to github.

1. Submit the link to your github repo in sakai.
