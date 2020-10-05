<!--THIS FILE IS AUTOGENERATED: DO NOT EDIT-->
<!--See https://github.com/dimagi/commcare-cloud/blob/master/changelog/README.md for instructions-->
# 37. ES upgrade from 2.4.6 to 7.9.1

**Date:** 2020-10-01

**Optional per env:** _required on all environments_


## CommCare Version Dependency
The following version of CommCare must be deployed before rolling out this change:
[9bd24b8f](https://github.com/dimagi/commcare-hq/commit/9bd24b8ff26266e65be2f093796b64116da72dac)


## Change Context
This change upgrades Elasticsearch from 2.4.6 to 7.9.1 version.
CommCare HQ releases after **Todo; decide** will not continue to support Elasticsearch 2.4.6,
so this change must be applied before then.

## Details
As part of our ongoing effort to keep CommCare HQ up to date with the latest tools and
libraries we have updated Elasticsearch.

This upgrade requires downtime and additional hardware resources in most cases.  It is recommended that you read this document fully before deciding on an upgrade method.

### Planning your upgrade

The upgrade has following three stages. 

 1. **Setup**: A new separate Elasticsearch 7 cluster is setup (could be a monolith or a set of VMs, depending on your deployment).
 3. **Reindex**: The data that exists in Elasticsearch 2 is reindexed into the new Elasticsearch 7 cluster. The duration of the reindex depends on your Elasticsearch index sizes. 
 4. **Switchover**: CommCareHQ is switched over to use the new Elasticsearch 7 cluster. This requires a short downtime of less than an hour.

Two different methods of reindexing are made available that balance the amount of downtime required to upgrade vs the ease of upgrade. 

 1. **Reindex and Switchover in one go**: The entire application is stopped and a preindex management command is run to reindex data into Elasticsearch 7. This method is recommended
    - If your Elasticsearch form and case index sizes are less than 20GB (excluding replicas) each. You may find the index size from Elasticsearch [indices stats API](https://www.elastic.co/guide/en/elasticsearch/reference/2.4/indices-stats.html) or using a tool like [ElasticHQ](https://www.elastichq.org/)
    - Or if you do lack additional temporary hardware resources.
 2. **Reindex and Switchover separately**: Majority of the data is reindexed without downtime and the offset data is reindexed with a short downtime just prior to switchover. The offset data is that which is freshly indexed into Elasticsearch 2 from the application while the ES2 data is being reindexed into ES7. 
    - This method is recommended if your Elasticsearch form and case index sizes are more than 20GB (excluding replicas) each. 

In either cases, it is recommended to do a dry-run reindex to surface any issues related to your data and get an estimate of downtime for actual switchover. Notes on how this can be done are provided under each method.

## Steps to update


- [Setup](#setup)
- [Reindex and Switchover in one go (Method 1)](#reindex-and-switchover-in-one-go)
- [Reindex and Switchover separately. (Method 2)](#reindex-and-switchover-separately)
  - [Reindex](#reindex)
  - [Switchover](#switchover)
- [Appendix](#appendix)
  - [Note 1: To get stats on inserted_at field for your data](#note-1-to-get-stats-on-inserted_at-field-for-your-data)
  - [Note 2: Resetting ES7 indices after dry-run](#note-2-resetting-es7-indices-after-dry-run)
  - [Note 3: Additional setup steps to run ES2 and ES7 on same machines](#note-3-additional-setup-steps-to-run-es2-and-es7-on-same-machines)
  - [Note 4: Upgrade without extra hardware resources](#note-4-upgrade-without-extra-hardware-resources)




### Setup

1. Provision and setup a separate ES7 cluster of the same size or larger than your current ES2 cluster by doing the below using commcare-cloud
    - Replace `elasticsearch` group in your inventory with new hosts 
    - Update below settings in your public.yml file
      ```
      elasticsearch_version: 7.9.1
      elasticsearch_cluster_name: es7_<cluster_name> # must start with es7_ prefix
      elasticsearch_download_sha256: a1b43a6e29d3ca91d08366f64007ce812646e4775524214f66330d447a4c6e3c
      remote_es2_host: <Link to your existing ES2 cluster in http://host:port format. This will be used on the ES7 cluster to allow permissions for remote reindexing>
      ```
    - Run commcare-cloud ansible deploy on new hosts
      ```
      commcare-cloud <env> deploy-stack --limit=<new_es7_nodes> --branch=<git-upgrade-branch>
      ```
        - If you do not have buffer resources for a separate ES7 cluster or if you have a monolith, you can setup ES7 on any of your existing machines other than ES2.
        - If you have to setup ES7 exactly on the same machines as your current ES2 cluster, follow additional steps in Appendix [Note 3](#note-3-additional-setup-steps-to-run-es2-and-es7-on-same-machines) before proceeding to run ES2 and ES7 on same machines. This requires at least 60% free storage/memory
        - If you do not have extra 60% free storage/memory, follow Appendix [Note 4](#note-4-upgrade-without-extra-hardware-resources) to upgrade to ES7 which requires full downtime and is a much slower process (suitable only for very small monoliths)
2. Setup a private release from which to run reindex commands. Note down the release directory and the web worker where the release is setup
    ```
    commcare-cloud <env> fab setup_limited_release:keep_days=10
    ```
3. Manually update below settings in the above release directory's `localsettings.py` file
      ```
      ELASTICSEARCH_MAJOR_VERSION = 7
      ELASTICSEARCH_HOSTS = [
      ...new ES7 hosts...
      ]
      ELASTICSEARCH_PORT = 9200 # or whichever ES7 port you are using
     ```
4. Intialize HQ indices on ES7 cluster by running this management command from the release directory created above
      ```
      ./manage.py initialize_es_indices
      ```
      - You can use an Elasticsearch monitoring tool such as https://www.elastichq.org/ to make sure the indices are created with expected number of shards. The tool will be useful to check reindex progress also.

Once you have the cluster setup, if your index sizes are small you can [reindex and switchover in one go](#reindex-and-switchover-in-one-go) which is easy but requires full application downtime. If your index sizes are large you can reindex form/case indices [first without downtime and then proceed to switchover](#reindex-and-switchover-separately) with a small downtime.



### Reindex and Switchover in one go

If your indices are smaller than 20GB (excluding replicas) each, the reindex and switchover phases can be combined to simplify the upgrade process. 

This requires downtime. To estimate the downtime required and to surface any data issues you should run step 4 below in `dry-run` mode before the actual upgrade. (See Appendix Note 2, to reset the cluster before actual upgrade)

Follow below instructions to reindex data into the new ES7 cluster and switchover HQ to use Elasticsearch 7

1. Start downtime
    ```
    commcare-cloud <env> downtime start
    ```
2. At this point, your ES2 hosts should not be getting any HTTP requests.
    - You can confirm this by doing a tcpdump on Elasticsearch port on any of your Elasticsearch 2 hosts
        ```
        tcpdump -A -s 0 'tcp port 9200 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'_
        ```
3. Reindex by running below command for each of the index in`['forms', 'cases', 'users', 'domains', 'apps', 'groups', 'sms', 'report_cases', 'report_xforms', 'case_search']`
    ```
    ./manage.py reindex_es_from_remote <index_name> <es2_host>
    ```
    - These can be run concurrently in different tmux/screen sessions
    - Note that these management commands are to be run from the release that was created in Setup phase, which is pointing to Elasticsearch 7
    - See command help for additional options
4. At this point, all of the Elasticsearch 2 data should be in Elasticsearch 7.
    - You can confirm this by running this command to display the number of documents in both ES clusters:
        ```
        ./manage.py reindex_es_from_remote --print_index_size
        ```
    - Alternatively you may use an ES monitoring tool such as [ElasticHQ](https://www.elastichq.org/)
5. Run below command to update Elasticsearch settings on ES7 indices for usage
    ```
    ./manage.py initialize_es_indices --set-for-usage
    ```
6. Update HQ config to point to Elasticsearch 7.
    - Update below settings in you env's public.yml in addition to other setting changes made in git-upgrade-branch in the Setup phase
        ```
        ELASTICSEARCH_MAJOR_VERSION: 7
        ```
    - Update config
        ```
        commcare-cloud <env> update-config --branch=<git-upgrade-branch>
        ```
    - Merge the git-upgrade-branch into master  
7. Make sure that HQ is able to connect to Elasticsearch 7
    ```
    commcare-cloud <env> django-manage check_services
    ```
8. End downtime
    ```
    commcare-cloud <env> downtime end
    ```
That's it, you are running HQ on Elasticsearch 7!



### Reindex and Switchover separately

If your form/case index sizes are large, you can follow this method to first reindex without downtime and then switchover with downtime.

#### Reindex

- Reindex phase involves reindexing data from Elasticsearch 2 cluster into new Elasticsearch 7 cluster.
- Form and Case indices can be indexed in background while the HQ application continues to be live.
- This reindex can be performed based on the `inserted_at` date field up to a day before switchover. 
- Using this parameter multiple reindex processes can be run concurrently for different date ranges enabling you to reindex very large indices.
- The smaller offset data inserted newly during and after the reindex phase can be reindexed  during the last phase of switchover with downtime. This enables minimising the overall downtime required to upgrade drastically. 

Use the following process to reindex all of the data up to a day or two before switchover and proceed to the switchover section after that.

 1. The reindex management command should be run from the release that's setup in step 2 of Setup phase above which is pointing to Elasticsearch 7. 
    - To activate virtual env for reindex commands, SSH into the web worker from Step 2 of Setup phase and do below
        ```
        tmux # optional
        sudo -iu cchq
        cd <releases_dir_from_setup_phase>
        source python_env-3.6/bin/activate
        grep ELASTICSEARCH localsettings.py # to make sure this points to ES7
        ```
 2. Reindex all of form/case data up to a planned switchover date
    - Below is an example management command that reindexes forms index from Elasticsearch 2 cluster at `http://es2_cluster:9200` for a specific date range
      ```
        # to get help on usage
      ./manage.py reindex_es_from_remote --help
      # an example reindex command
        ./manage.py reindex_es_from_remote forms http://es2_cluster:9200 --start_date=YYYY-MM-DD --end_date=YYYY-MM-DD 
      ``` 
    - Multiple reindex commands with different date ranges can be run in separate tmux/screen session to maximize reindex throughput.
    - To efficiently split the reindex commands you may see Appendix [Note 1](#note-1-to-get-stats-on-inserted_at-field-for-your-data) below to get stats on `inserted_at` field for your form/case data.
    - The start_date and end_date are optional and inclusive.
    - If the dates are not specified all the data is reindexed in a single command.
    - The command doesn't support pause/resume, so each time it reindexes all the data in the specified date range again, even if the data is already reindexed.
    - Make a note of the date on which you have initiated your first form/case reindex. This is required in the switchover phase.
 3. Below is a sequence of commands for an example scenario 
    ```
    # reindex all of form data up to Jan 2016.
    ./manage.py reindex_es_from_remote forms http://es2_cluster:9200 --end_date=2016-01-01
    # reindex form data from Jan 2016 to October 2020
    ./manage.py reindex_es_from_remote forms http://es2_cluster:9200 --start_date=2016-01-01 --end_date=2020-10-01
    # reindex form data added newly since October 1st 2020 (this can be done in the switchover phase explained next) 
    ./manage.py reindex_es_from_remote forms http://es2_cluster:9200 --start_date=2020-10-01
    ```

Forms and cases that are deleted during the reindex continue to stay in ES7. These deletes will be reprocessed in the switchover phase. This step requires a date after which the deletes are to be reprocessed, which will be the date on which you have initiated your first form/case reindex. So, make a note of that date.

You can now proceed to switchover.

#### Switchover

The switchover phase requires downtime to reindex rest of ES indices and offset form/case index data. The duration depends on the amount of data in other indices and the offset data.

1. Start downtime
    ```
    commcare-cloud <env> downtime start
    ```
2. At this point, your ES2 hosts should not be getting any HTTP requests.
    - You can confirm this by doing a tcpdump on Elasticsearch port on any of your Elasticsearch 2 hosts
      ```
      tcpdump -A -s 0 'tcp port 9200 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'_
      ```
3. Reindex offset form/case data inserted after the Reindex phase.
    ```
    ./manage.py reindex_es_from_remote forms http://es2_cluster:9200 --start_date=<last_reindex_date>
    ./manage.py reindex_es_from_remote cases http://es2_cluster:9200 --start_date=<last_reindex_date>
    ```
    - last_reindex_date is the date up to which you have already reindexed in the Reindex phase
    - This and subsequent commands are to be run from the release directory that's pointing to Elasticsearch 7
4. Reindex rest of the indices by running below command for each of the index in `['users', 'domains', 'apps', 'groups', 'sms', 'report_cases', 'report_xforms', 'case_search']`
    ```
    ./manage.py reindex_es_from_remote <index_name> <es2_host>
    ```
    These can be run concurrently in different tmux/screen sessions
5. Reprocess form and case deletes since your first reindex from the reindex phase.
    ```
    ./manage.py reprocess_form_case_es_deletes --from=<date-of-when-migration-is-initiated>
    ```
6. At this point, all of the Elasticsearch 2 data should be in Elasticsearch 7. You can confirm this by running the `reindex_es_from_remote` management command with `--print_index_size` option which displays number of documents in both the ES clusters. 
7. Run below command to update Elasticsearch settings on ES7 indices for usage
    ```
    ./manage.py initialize_es_indices --set-for-usage
    ```
8. Update HQ services to point to Elasticsearch 7.
    - Update below settings in your env's public.yml in addition to other setting changes made in git-upgrade-branch from Setup phase
      ```
      ELASTICSEARCH_MAJOR_VERSION: 7
      ```
    - Update config
      ```
      commcare-cloud <env> update-config --branch=<git-upgrade-branch>
      ```
    - Merge the git-upgrade-branch into master  
9. Make sure that HQ is able to connect to Elasticsearch 7
    ```
    commcare-cloud <env> django-manage check_services
    ```
10. End downtime
    ```
    commcare-cloud <env> downtime end
    ```
That's it, you are running HQ on Elasticsearch 7!

**Getting Estimate for switchover**

If you have large cluster you can get an estimate of downtime required for switchover. In most cases this is not necessary.

The major downtime will be required in steps 3, 4 and 5. You might already have an estimate for step 4 based on the reindex phase. These can be run prior to get an estimate, but they will need to be rerun during actual switchover as well

To get the estimate:
- execute step 4 to get estimate for reindexing indices other than form/case. These indices need to be reset before actual switchover. You can see Appendix [Note 2](#note-2-resetting-es7-indices-after-dry-run)
- execute step 3 and 5 to get estimate for offset form/case data and reprocessing form/case deletes, but these shouldn't be reset since this affects already reindexed data from reindex phase


### Appendix 

##### Note 1 To get stats on inserted_at field for your data
To split your form/case reindex commands optimally based on `inserted_at` you can get oldest `inserted_at` date and monthly counts based on it by running below code on a Django shell pointing to ES2 (not ES7) .

```
from corehq.apps.es.aggregations import MaxAggregation, MinAggregation, DateHistogram
from corehq.apps.es.cases import CaseES
from corehq.apps.es.forms import FormES

# oldest inserted_at date
CaseES().aggregation(MinAggregation('in', 'inserted_at')).run()
FormES().aggregation(MinAggregation('in', 'inserted_at')).run()

# monthly count aggregation on inserted_at
CaseES().aggregation(DateHistogram('in', 'inserted_at', 'month')).run()
FormES().aggregation(DateHistogram('in', 'inserted_at', 'month')).run()
```

##### Note 2 Resetting ES7 indices after dry run

Note that this clears all the data and recreates blank indices so make sure you are running it from a release folder that is pointing to the ES7 cluster and NOT the ES2 cluster

- To reset all ES7 indices, run below command from the release pointing to ES7
  ```
  ./manage.py initialize_es_indices --reset
  ```
- You can specify an index via `--index` option to reset a single index.

##### Note 3 Additional setup steps to run ES2 and ES7 on same machines

Commcare-cloud doesn't support running ES2 and ES7 on same hosts. Some manual steps are required to run ES2 and ES7 run on same machines.

- Make sure that you have at least 60% free memory and at least 60% free storage 
- Update below settings in public.yml in the git es7-upgrade-branch 
    ```
    elasticsearch_tcp_port: must be other than 9300 or your corresponding ES2 tcp port
    elasticsearch_http_port: must be other than 9200 or your corresponding ES2 http port
    elasticsearch_memory: available free memory on your machines min 4GB
    ```
- Create an elasticsearch7 systemd/upstart service manually.
  - Stop the application with `commcare-cloud <env> downtime start`
  - Stop Elasticsearch `commcare-cloud <env> service elasticsearch stop`
  - Make a backup of elasticsearch 2 upstart conf file `cp /etc/init/elasticsearch.conf /etc/init/elasticsearch2.backup`  or systemd file `cp /etc/systemd/system/elasticsearch.service /etc/systemd/system/elasticsearch2.backup`
  - Deploy ES7 with `commcare-cloud <env> deploy-stack --limit=elasticsearch --branch=<git-upgrade-branch>` This will delete ES2 systemd/upstart service but keep ES2 as is
    - (If you are using upstart) Create elasticsearch7 service `cp /etc/init/elasticsearch.conf /etc/init/elasticsearch7.conf`
    - (If you are using Systemd)  Create elasticsearch7 service `cp /etc/systemd/system/elasticsearch.service /etc/systemd/system/elasticsearch7.service` and `systemctl daemon-reload"`
  - Restore ES2 service file 
    - `cp /etc/systemd/system/elasticsearch2.backup /etc/systemd/system/elasticsearch.service`  and `systemctl daemon-reload"` 
    - Or for upstart `cp /etc/init/elasticsearch2.backup /etc/init/elasticsearch.conf`
  - Start ES2 by `sudo service elasticsearch start`
  - Start ES7 by `sudo service elasticsearch7 start`

You may now proceed from rest of the steps from Setup phase


##### Note 4 Upgrade without extra hardware resources

If you do not have access to any spare resources and would like to simply replace Elasticsearch 2 with 7, you can delete the existing Elasticsearch 2 cluster and deploy Elasticsearch 7 cluster. This will require downtime while the data is being indexed into Elasticsearch 7. We strongly recommend against this approach as it could be slower to reindex data from HQ to Elasticsearch 7 during which your application needs to be down. Below are the steps to do this.

1. Update below settings in your public.yml file
    ```
    elasticsearch_version: 7.9.1
    elasticsearch_cluster_name: es7_<cluster_name> # must start with es7_ prefix
    elasticsearch_download_sha256: a1b43a6e29d3ca91d08366f64007ce812646e4775524214f66330d447a4c6e3c
    ELASTICSEARCH_MAJOR_VERSION: 7
    ```
  - Run `commcare-cloud <env> deploy-stack --limit=<new_es7_nodes>` to deploy ES7
2. Start downtime `commcare-cloud <env> downtime start`
3. Make sure you have a [backup of Elasticsearch](http://dimagi.github.io/commcare-cloud/commcare-cloud/backup.html#elasticsearch-snapshots) 2 cluster and delete the data folder `/opt/data/elasticsearch2.4.6`
4. Reindex data using `commcare-cloud <env> django_manage preindex_everything`
5. After the preindex is finished end the downtime using `commcare-cloud <env> downtime stop`