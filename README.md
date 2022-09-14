# kong-demo
kong-demo


### Install Docker
- You can install docker on your system by visiting https://docs.docker.com/engine/install/
### Docker Swarm Initilization
- Initializing docker swarm
```
docker swarm init

# Output
Swarm initialized: current node (w59qus3pt0wcqk604dtbk08cv) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-0zqcnonxkzyk6upa5nw97zn2xfp3egqqfzmv972lpnq5cvmy64-c1nquxdneqckbg3pvlru9b26h 192.168.56.10:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

### Docker Network Creation
- Creating a network named swarm of type overlay so that all services run on same network and area able to communicate using service name
```
docker network create --driver overlay swarm

# Output
lsx4bp0hodv8skttfxcdoyi2a
```

### Postgres Container
- Create a local directory to hold the postgres data so that after system reboot, we dont have to recreate the data
```
mkdir postgres_data
```
- Deploy postgres container
```
docker stack deploy -c postgres.yml postgres
```
- Check postgres is running before proceeding further
```
docker service ls

# Output
ID             NAME                MODE         REPLICAS   IMAGE          PORTS
n4bpackm6yan   postgres_postgres   replicated   1/1        postgres:9.6   *:5432->5432/tcp
```
- Create a kong database
```
CONTAINER_ID=$(docker ps | grep postgres | awk '{print $1}')
docker exec -it -u postgres $CONTAINER_ID bash -c 'psql -c "CREATE DATABASE kong;"'
```
- Verify database exists
```
docker exec -it -u postgres $CONTAINER_ID bash -c 'psql -c "\l"' | grep kong

# Output
kong      | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
```

### Create the Kong Container
- Run the migration first. This is required only once on a new database
```
docker run --rm --network host --name kong-migration -e "KONG_DATABASE=postgres" -e "KONG_PG_DATABASE=postgres" -e "KONG_PG_HOST=127.0.0.1" -e "KONG_PG_PASSWORD=password" -e "KONG_PG_USER=postgres" kong:0.14.1 kong migrations up --vv

# Output
2022/09/14 08:14:12 [info] 67 migrations ran
2022/09/14 08:14:12 [verbose] migrations up to date
```
- Deploy kong
```
docker stack deploy -c kong.yml kong
```

- Check kong and postgres are running before proceeding further
docker service ls
```
# Output
ID             NAME                    MODE         REPLICAS   IMAGE                        PORTS
wvmkz4a97stk   kong_kong               replicated   1/1        kong:0.14.1                  *:8000-8001->8000-8001/tcp
n4bpackm6yan   postgres_postgres       replicated   1/1        postgres:9.6                 *:5432->5432/tcp
```


### Echo Container
- Deploy echo container
```
docker stack deploy -c headerecho.yml headerecho
```
- Check headerecho, kong and postgres are running before proceeding further
```
docker service ls

# Output
ID             NAME                    MODE         REPLICAS   IMAGE                        PORTS
shxuvkc2yiio   headerecho_headerecho   replicated   1/1        keshavprasad/headerecho:v3   *:4000->4000/tcp
wvmkz4a97stk   kong_kong               replicated   1/1        kong:0.14.1                  *:8000-8001->8000-8001/tcp
n4bpackm6yan   postgres_postgres       replicated   1/1        postgres:9.6                 *:5432->5432/tcp
```

### Accessing Kong
- Accessing Data Plane
```
curl localhost:8000
```
- Accessing Control Plane
```
curl localhost:8001
```

### Creating a new API in Kong
- Lets try to access an API with path as /echo
```
curl localhost:8000/echo

# Output
{"message":"no route and no API found with those values"}
```
- We dont have any API yet in kong with the path /echo
- Lets add an API
- We are creating an api named headerecho
- The URL to access the api is /echo
- If we get a request ending with /echo, the request should be sent to a backend named headerecho_headerecho on port 4000
- We can use the docker service name headerecho_headerecho to access the application from another container
- The service names and ports are shown in previous command when you ran docker service ls

```
curl -X POST localhost:8001/apis --data "name=echo" --data "upstream_url=http://headerecho_headerecho:4000" --data "uris=/echo"

# Output
{"created_at":1663144056650,"strip_uri":true,"id":"42373e62-b83a-4746-a170-a3f7335a4dc7","name":"headerecho","http_if_terminated":false,"preserve_host":false,"upstream_url":"http:\/\/headerecho_headerecho:4000","uris":["\/echo"],"upstream_connect_timeout":60000,"upstream_send_timeout":60000,"upstream_read_timeout":60000,"retries":5,"https_only":false}
```
- Now lets access the API
```
curl localhost:8000/echo

# Output
GET /1 HTTP/1.1
Host: headerecho_headerecho:4000
Accept: */*
Connection: keep-alive
User-Agent: curl/7.68.0
X-Forwarded-For: 10.0.0.2
X-Forwarded-Host: localhost
X-Forwarded-Port: 8000
X-Forwarded-Proto: http
X-Real-Ip: 10.0.0.2
```

### Lets add authentication to API
- We are adding a plugin named jwt to the echo API
- The path /apis is used to interact with apis. This is an internal endpoint in kong
- The name headerecho is the API name we used while creating the API
- We are telling kong to add a plugin named jwt to out echo API
```
curl -XPOST localhost:8001/apis/headerecho/plugins --data "name=jwt"

# Output
{"created_at":1663144636000,"config":{"secret_is_base64":false,"key_claim_name":"iss","cookie_names":{},"maximum_expiration":0,"anonymous":"","run_on_preflight":true,"uri_param_names":["jwt"]},"id":"db905ce4-bc04-4859-935a-411e59870a73","name":"jwt","api_id":"42373e62-b83a-4746-a170-a3f7335a4dc7","enabled":true}
```
- Lets try to access the API now
```
curl localhost:8000/echo

# Output
{"message":"Unauthorized"}
```
- We cannot access this API without an API token after adding the jwt plugin

### Create a Consumer who will be able to access the API
- The path /consumer is used to interact with consumers. This is an internal endpoint in kong
- We are creating a user / consumer named ram
```
curl -XPOST localhost:8001/consumers --data "username=ram"

# Output
{"custom_id":null,"created_at":1663144816,"username":"ram","id":"6a14af33-55ec-4352-8ca5-aa63796b3c39"}
```

### Create a JWT token for a consumer
- We are creating a jwt credential for the consumer named ram 
- We can assign any value for the key, but they need to be unique per consumer
- In this case, lets name the key as ramskey
- Using the key and consumer, we will be able to identify who is accessing the applicaton, how many times they are accessing etc, if we setup API monitoring

```
curl -XPOST localhost:8001/consumers/ram/jwt -H "Content-Type: application/x-www-form-urlencoded" --data "key=ramskey"

# Output
{"created_at":1663145067000,"id":"b0c99019-1671-418a-ba3b-c24a1aa04d06","algorithm":"HS256","key":"ramskey","secret":"3nSpeftPAKtOj1SoHtLh6191zT8ng0AP","consumer_id":"6a14af33-55ec-4352-8ca5-aa63796b3c39"}
```

### Creating the token from key and secret
- Go to [jwt.io](https://jwt.io/) and paste the below in `PAYLOAD` section
```
{
  "iss": "ramskey"
}
```
- Paste the `secret` value, shown in the previous step, in [jwt.io](https://jwt.io/) `VERIFY SIGNATURE` section. Leave the field `secret base64 encoded` unchecked
- Copy the JWT token displayed
- This is your API token
- In my case the API token is
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJyYW1za2V5In0.wcWJHCFUz9bxQ55sJA7ZK7JK2IxlGDcyiLjg3ETYLoA
```
- Lets set this value as a variable
```
export APITOKEN=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJyYW1za2V5In0.wcWJHCFUz9bxQ55sJA7ZK7JK2IxlGDcyiLjg3ETYLoA
```
### Access the API using api token
- We can access the API by passing the `Authorization` and the API bearer key as `Bearer TOKEN`
```
curl -H "Authorization: Bearer $APITOKEN" localhost:8000/echo

# Output
GET / HTTP/1.1
Host: headerecho_headerecho:4000
Accept: */*
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJyYW1za2V5In0.wcWJHCFUz9bxQ55sJA7ZK7JK2IxlGDcyiLjg3ETYLoA
Connection: keep-alive
User-Agent: curl/7.68.0
X-Consumer-Id: 6a14af33-55ec-4352-8ca5-aa63796b3c39
X-Consumer-Username: ram
X-Forwarded-For: 10.0.0.2
X-Forwarded-Host: localhost
X-Forwarded-Port: 8000
X-Forwarded-Proto: http
X-Real-Ip: 10.0.0.2
```
- We have now protected our API
- General public cannot use our API without an api token

### Lets add rate limiting for our API
- We want to ensure a user (ram) can access our API only 2 times per minute
- Lets enable the rate limiting plugin for our API
- We will rate limit based on `consumer` and store the count of how many times one has accessed in kong's `local` memory
```
curl -XPOST localhost:8001/apis/headerecho/plugins --data "name=rate-limiting" --data "config.minute=2" --data "config.policy=local" --data "config.limit_by=consumer"

# Output
{"created_at":1663146200000,"config":{"hide_client_headers":false,"minute":2,"policy":"local","redis_database":0,"redis_timeout":2000,"redis_port":6379,"limit_by":"consumer","fault_tolerant":true},"id":"973a415b-0c5c-4d8e-b899-bfbefa7dc2c9","name":"rate-limiting","api_id":"42373e62-b83a-4746-a170-a3f7335a4dc7","enabled":true}
```
- Lets try to access the API more than 2 times in a minute
- Call the API 3 times
```
curl -H "Authorization: Bearer $APITOKEN" localhost:8000/echo
curl -H "Authorization: Bearer $APITOKEN" localhost:8000/echo
curl -H "Authorization: Bearer $APITOKEN" localhost:8000/echo

# Output
{"message":"API rate limit exceeded"}
```
- We have now implemented rate limiting and DDOS attacks are now minimzed


### Lets add a group to our API (ACL - Access control list)
- We want to group the API on some logic
- Maybe adming apis, teacher apis, student apis etc
- We want to ensure that a student cannot invoke teacher api using the API token
- Lets add a group named `teacher` for our api
- Get the API `id` first
```
curl localhost:8001/apis/headerecho

# Output
{"created_at":1663144056650,"strip_uri":true,"id":"42373e62-b83a-4746-a170-a3f7335a4dc7","name":"headerecho","http_if_terminated":false,"https_only":false,"upstream_url":"http:\/\/headerecho_headerecho:4000","uris":["\/echo"],"preserve_host":false,"upstream_connect_timeout":60000,"upstream_read_timeout":60000,"upstream_send_timeout":60000,"retries":5}
```
- Copy the `id` value from above command use it in the next command
- We are adding another kong plugin named `acl` which will help us achieving fine grain control in accessing our APIs
```
curl -X POST http://localhost:8001/plugins --data "name=acl"  --data "config.whitelist=teacher" --data "config.hide_groups_header=true" --data "api_id=42373e62-b83a-4746-a170-a3f7335a4dc7"

# Output
{"created_at":1663146848000,"config":{"whitelist":["teacher"],"hide_groups_header":true},"id":"81544ee7-6c5c-4a1d-a9c5-134f6764df6f","name":"acl","api_id":"42373e62-b83a-4746-a170-a3f7335a4dc7","enabled":true}
```
- Now we have added a group named `teacher` to the API named `headerecho`
- Lets just assume `Ram` is a `student` and he will try to access this API
```
curl -H "Authorization: Bearer $APITOKEN" localhost:8000/echo

# Output
{"message":"You cannot consume this service"}
```
- Now after 6 months, `Ram` completed the course and joined back as a teacher
- Now we need to provide the access to `Ram`
- Lets adding the `teacher` group to `Ram`
```
curl -XPOST localhost:8001/consumers/ram/acls --data "group=teacher"

# Output
{"group":"teacher","created_at":1663147259000,"id":"3e4bea6a-fabb-422c-867e-68a19681d80c","consumer_id":"6a14af33-55ec-4352-8ca5-aa63796b3c39"}
```
- Lets try to access the API now using same token
```
curl -H "Authorization: Bearer $APITOKEN" localhost:8000/echo

# Output
GET / HTTP/1.1
Host: headerecho_headerecho:4000
Accept: */*
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJyYW1za2V5In0.wcWJHCFUz9bxQ55sJA7ZK7JK2IxlGDcyiLjg3ETYLoA
Connection: keep-alive
User-Agent: curl/7.68.0
X-Consumer-Id: 6a14af33-55ec-4352-8ca5-aa63796b3c39
X-Consumer-Username: ram
X-Forwarded-For: 10.0.0.2
X-Forwarded-Host: localhost
X-Forwarded-Port: 8000
X-Forwarded-Proto: http
X-Real-Ip: 10.0.0.2
```