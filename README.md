How to set up
1. install docker engine

2. Using docker-compose
Create a file named “docker-compose.yml” and paste the following:

version: "3.10"

services:
  redis:
    image: redis:alpine
    restart: unless-stopped
    healthcheck:
      test: redis-cli ping | grep PONG
    deploy:
      resources:
        limits:
          cpus: '0.2'
          memory: 128M
    security_opt:
      - no-new-privileges:true

  rabbit-mq:
    image: rabbitmq:3.10-management-alpine
    restart: unless-stopped
    healthcheck:
      test: rabbitmq-diagnostics -q ping
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 525M
    security_opt:
      - no-new-privileges:true

  prometheus:
    image: prom/prometheus:latest
    restart: unless-stopped
    ports:
      - '9090:9090'
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - monitoring_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'
    depends_on:
      - jasmin
    deploy:
      resources:
        limits:
          cpus: '0.2'
          memory: 128M
    security_opt:
      - no-new-privileges:true

  grafana:
    image: grafana/grafana
    restart: unless-stopped
    ports:
      - 3000:3000
    environment:
      GF_INSTALL_PLUGINS: "grafana-clock-panel,grafana-simple-json-datasource"
    volumes:
      # These mount points should be copied from https://github.com/jookies/jasmin/tree/master/docker/grafana
      - ./provisioning/datasources:/etc/grafana/provisioning/datasources:ro
      - ./provisioning/dashboards:/etc/grafana/provisioning/dashboards:ro
      - ./dashboards:/opt/grafana-dashboards:ro
      - monitoring_data:/var/lib/grafana
    depends_on:
      - prometheus
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
    security_opt:
      - no-new-privileges:true

  jasmin:
    image: jookies/jasmin:latest
    restart: unless-stopped
    ports:
      - 2775:2775
      - 8990:8990
      - 1401:1401
    depends_on:
      redis:
        condition: service_healthy
      rabbit-mq:
        condition: service_healthy
    environment:
      REDIS_CLIENT_HOST: redis
      AMQP_BROKER_HOST: rabbit-mq
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 256M
    security_opt:
      - no-new-privileges:true

volumes:
  monitoring_data: { }



3. Then spin it:

docker-compose up -d
This command will pull latest jasmin v0.10, latest redis and latest rabbitmq images to your computer:

4. # docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
jasmin              latest              0e4cf8879899        36 minutes ago      478.6 MB
Jasmin is now up and running:

# docker ps
CONTAINER ID   IMAGE                 COMMAND                  CREATED         STATUS         PORTS                                                                    NAMES
1a9016d298bf   jookies/jasmin:0.10   "/docker-entrypoint.…"   3 seconds ago   Up 2 seconds   0.0.0.0:1401->1401/tcp, 0.0.0.0:2775->2775/tcp, 0.0.0.0:8990->8990/tcp   jasmin
af450de4fb95   rabbitmq:alpine       "docker-entrypoint.s…"   5 seconds ago   Up 3 seconds   4369/tcp, 5671-5672/tcp, 15691-15692/tcp, 25672/tcp                      rabbitmq
c8feb6c07d94   redis:alpine          "docker-entrypoint.s…"   5 seconds ago   Up 3 seconds   6379/tcp                                                                 redis
Note

You can play around with the docker-compose.yml to choose different versions, mounting the configs outside the container, etc …



5. seting up jasmin
--------------------------
Connect to jCli console through telnet (telnet 127.0.0.1 8990) using jcliadmin/jclipwd  and make the following configurations for
mo,mt smpp&http connectors,
mo,mt filters,
mo,mt routes,
groups,
users

telnet 127.0.0.1 8990

Authentication required.

Username: jcliadmin
Password:
Welcome to Jasmin console
Type help or ? to list commands.

Add airtel Connector
------------------------

jcli : smppccm -a
> cid AIRTEL_CONNECTOR
> host messaging.airtelkenya.com
> port 9001
> username sozuriSMPP
> password Zain@124
> submit_throughput 20
> ok
Successfully added connector [AIRTEL_CONNECTOR]

Add Safaricom Connector
------------------------

jcli : smppccm -a
> cid SAFARICOM_CONNECTOR
> host 196.201.213.135
> port 2775
> username sozuriSMPP
> password Zain@124
> submit_throughput 20
con_fail_delay 10
dlr_expiry 86400
coding 0
submit_throughput 1
elink_interval 10
bind_to 30
port 2775
con_fail_retry yes
password password
src_addr None
bind_npi 1
addr_range None
dst_ton 1
res_to 60
def_msg_id 0
priority 0
con_loss_retry yes
dst_npi 1
validity None
requeue_delay 120
host 127.0.0.1
src_npi 1
trx_to 300
logfile /var/log/jasmin/default-Demo.log
systype
cid Demo
loglevel 20
bind transmitter
proto_id None
con_loss_delay 10
bind_ton 0
pdu_red_to 10
src_ton 2
> ok


add http connector for MO
--------------------------
httpccm -a
Adding a new Httpcc: (ok: save, ko: exit)
> url http://10.0.0.131/receive-sms/mo
> method GET
> cid SOZURI_HTTP_MO_01
> ok


start connectors
---------------------
smppccm -1 AIRTEL_CONNECTOR
smppccm -1 SAFARICOM_CONNECTOR
smppccm -1 SOZURI_HTTP_MO_01


check
---------
smppccm --list
httpccm -s HTTP-01
httpccm -l



to customers downstream, from telco, inbound us, from subscriber
MO Route is used to route inbound messages (SMS MO) through two possible channels: http and smpps (SMPP Server).

create an MO Route to send sms to downstream SMPP sample customer

StaticMORoute to a SMPP Server user (user_1):
------------------------
jcli : morouter -a
Adding a new MO Route: (ok: save, ko: exit)
> type StaticMORoute
jasmin.routing.Routes.StaticMORoute arguments:
filters, connector
> order 15
> filters sozuri_mo_smpp_filter_1
> connector smpps(sozuri_smpp_customer)   #project here
> ok



create an MO Route to send sms to downstream Sozzuri HTTP mo route
-------------------------------------------

morouter -a
Adding a new MO Route: (ok: save, ko: exit)
> type StaticMORoute
jasmin.routing.Routes.StaticMORoute arguments:
filters, connector
> order 10
> filters sozuri_mo_http_filter_1
> connector http(SOZURI_HTTP_MO_01)
> ok



create an MT Route to send sms to upstream SMPP safaricom
---------------------

jcli : mtrouter -a
Adding a new MT Route: (ok: save, ko: exit)
> type StaticMTRoute
jasmin.routing.Routes.StaticMTRoute arguments:
filters, connector
> filters safaricom_mt_smpp_filter_1
> order 10
> connector smppc(SAFARICOM_CONNECTOR)
> rate 0.0
> ok

create an MT Route to send sms to upstream SMPP airtel
----------------

jcli : mtrouter -a
Adding a new MT Route: (ok: save, ko: exit)
> type StaticMTRoute
jasmin.routing.Routes.StaticMTRoute arguments:
filters, connector
> filters airtel_mt_smpp_filter_1
> order 10
> connector smppc(AIRTEL_CONNECTOR)
> rate 0.0
> ok



create group
---------------
group -a
Adding a new Group: (ok: save, ko: exit)
> gid marketing
> ok


smpp MT sender/user
------------------
user -a
Adding a new User: (ok: save, ko: exit)
> username foo
> password bar
> gid marketing
> uid foo
> smpps_cred authorization bind yes
> smpps_cred quota max_bindings 2
> mt_messaging_cred valuefilter src_addr ^JASMIN$
> mt_messaging_cred valuefilter dst_addr ^JASMIN$
> mt_messaging_cred quota balance 44.2
> mt_messaging_cred quota sms_count none
> mt_messaging_cred quota smpps_throughput 2
> smpps_cred authorization bind yes
> smpps_cred quota max_bindings 2
> ok


user -a
Adding a new User: (ok: save, ko: exit)
> username sozuri_smpp_customer
> password bar
> gid marketing
> uid sozuri_smpp_customer
> mt_messaging_cred valuefilter src_addr ^JASMIN$
> mt_messaging_cred quota balance 44.2
> mt_messaging_cred quota sms_count none
> mt_messaging_cred quota smpps_throughput 2
> ok



FILTERS:
--------------------
sozuri_mo_smpp_filter_1
sozuri_mo_http_filter_1
safaricom_mt_smpp_filter_1
airtel_mt_smpp_filter_1

SourceAddrFilter
----------------------------------
jcli : filter -a
Adding a new Filter: (ok: save, ko: exit)
> type sourceaddrfilter
> source_addr ^201356\d+
> fid sozuri_mo_smpp_filter_1
> ok


jcli : filter -a
Adding a new Filter: (ok: save, ko: exit)
> type sourceaddrfilter
> source_addr ^201303\d+
> fid sozuri_mo_http_filter_1
> ok


jcli : filter -a
Adding a new Filter: (ok: save, ko: exit)
> type sourceaddrfilter
> source_addr ^Sozuri$
#> destination_addr ^25472\d+
> fid safaricom_mt_smpp_filter_1
> ok

jcli : filter -a
Adding a new Filter: (ok: save, ko: exit)
> type sourceaddrfilter
#> source_addr ^Sozuri$
> destination_addr ^25473\d+
> fid airtel_mt_smpp_filter_1
> ok





stats
-------------


jcli : stats --users
jcli : stats --user sandra

jcli : stats --smppcs

jcli : stats --smppc MTN

jcli : stats --smppsapi

jcli : stats --httpapi







In order to reject a message, depending on the source of message (httpapi ? smpp server ? smpp client ?) the script must set smpp_status and/or http_status accordingly to the error to be returned back, here’s an error mapping table for smpp:

smpp_status Error mapping
Value	SMPP Status	Description
0	ESME_ROK	No error
1	ESME_RINVMSGLEN	Message Length is invalid
2	ESME_RINVCMDLEN	Command Length is invalid
3	ESME_RINVCMDID	Invalid Command ID
4	ESME_RINVBNDSTS	Invalid BIND Status for given command
5	ESME_RALYBND	ESME Already in Bound State
6	ESME_RINVPRTFLG	Invalid Priority Flag
7	ESME_RINVREGDLVFLG	Invalid Registered Delivery Flag
8	ESME_RSYSERR	System Error
265	ESME_RINVBCASTAREAFMT	Broadcast Area Format is invalid
10	ESME_RINVSRCADR	Invalid Source Address
11	ESME_RINVDSTADR	Invalid Dest Addr
12	ESME_RINVMSGID	Message ID is invalid
13	ESME_RBINDFAIL	Bind Failed
14	ESME_RINVPASWD	Invalid Password
15	ESME_RINVSYSID	Invalid System ID
272	ESME_RINVBCAST_REP	Number of Repeated Broadcasts is invalid
17	ESME_RCANCELFAIL	Cancel SM Failed
274	ESME_RINVBCASTCHANIND	Broadcast Channel Indicator is invalid
19	ESME_RREPLACEFAIL	Replace SM Failed
20	ESME_RMSGQFUL	Message Queue Full
21	ESME_RINVSERTYP	Invalid Service Type
196	ESME_RINVOPTPARAMVAL	Invalid Optional Parameter Value
260	ESME_RINVDCS	Invalid Data Coding Scheme
261	ESME_RINVSRCADDRSUBUNIT	Source Address Sub unit is Invalid
262	ESME_RINVDSTADDRSUBUNIT	Destination Address Sub unit is Invalid
263	ESME_RINVBCASTFREQINT	Broadcast Frequency Interval is invalid
257	ESME_RPROHIBITED	ESME Prohibited from using specified operation
273	ESME_RINVBCASTSRVGRP	Broadcast Service Group is invalid
264	ESME_RINVBCASTALIAS_NAME	Broadcast Alias Name is invalid
270	ESME_RBCASTQUERYFAIL	query_broadcast_sm operation failed
51	ESME_RINVNUMDESTS	Invalid number of destinations
52	ESME_RINVDLNAME	Invalid Distribution List Name
267	ESME_RINVBCASTCNTTYPE	Broadcast Content Type is invalid
266	ESME_RINVNUMBCAST_AREAS	Number of Broadcast Areas is invalid
192	ESME_RINVOPTPARSTREAM	Error in the optional part of the PDU Body
64	ESME_RINVDESTFLAG	Destination flag is invalid (submit_multi)
193	ESME_ROPTPARNOTALLWD	Optional Parameter not allowed
66	ESME_RINVSUBREP	Invalid submit with replace request (i.e. submit_sm with replace_if_present_flag set)
67	ESME_RINVESMCLASS	Invalid esm_class field data
68	ESME_RCNTSUBDL	Cannot Submit to Distribution List
69	ESME_RSUBMITFAIL	submit_sm or submit_multi failed
256	ESME_RSERTYPUNAUTH	ESME Not authorised to use specified service_type
72	ESME_RINVSRCTON	Invalid Source address TON
73	ESME_RINVSRCNPI	Invalid Source address NPI
258	ESME_RSERTYPUNAVAIL	Specified service_type is unavailable
269	ESME_RBCASTFAIL	broadcast_sm operation failed
80	ESME_RINVDSTTON	Invalid Destination address TON
81	ESME_RINVDSTNPI	Invalid Destination address NPI
83	ESME_RINVSYSTYP	Invalid system_type field
84	ESME_RINVREPFLAG	Invalid replace_if_present flag
85	ESME_RINVNUMMSGS	Invalid number of messages
88	ESME_RTHROTTLED	Throttling error (ESME has exceeded allowed message limits
271	ESME_RBCASTCANCELFAIL	cancel_broadcast_sm operation failed
97	ESME_RINVSCHED	Invalid Scheduled Delivery Time
98	ESME_RINVEXPIRY	Invalid message validity period (Expiry time)
99	ESME_RINVDFTMSGID	Predefined Message Invalid or Not Found
100	ESME_RX_T_APPN	ESME Receiver Temporary App Error Code
101	ESME_RX_P_APPN	ESME Receiver Permanent App Error Code
102	ESME_RX_R_APPN	ESME Receiver Reject Message Error Code
103	ESME_RQUERYFAIL	query_sm request failed
259	ESME_RSERTYPDENIED	Specified service_type is denied
194	ESME_RINVPARLEN	Invalid Parameter Length
268	ESME_RINVBCASTMSGCLASS	Broadcast Message Class is invalid
255	ESME_RUNKNOWNERR	Unknown Error
254	ESME_RDELIVERYFAILURE	Delivery Failure (used for data_sm_resp)
195	ESME_RMISSINGOPTPARAM	Expected Optional Parameter missing