# Troubleshooting a 502 Bad Gateway in the vProfile Project (Udemy DevOps Course)

While working through the vProfile project (Decoding DevOps course on Udemy), I hit a classic **502 Bad Gateway** error when accessing the app through the load balancer IP. Here's how I debugged it, step by step — sharing in case it helps someone else stuck at the same point.

## The Setup
- `web01` — Nginx reverse proxy
- `app01` — Tomcat running the vProfile app
- `rmq01` — RabbitMQ
- `db01` — MySQL
- `mc01` — Memcached

## Step 1: Is the app server even running?
```
sudo systemctl status tomcat
```
Tomcat was `active (running)` — so the service itself wasn't the problem. That ruled out the simplest cause.

## Step 2: Check the logs
```
sudo journalctl -u tomcat -n 50 --no-pager
```
This showed a wall of repeating errors:
```
ERROR org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer - Failed to check/redeclare auto-delete queue(s).
```
Tomcat was up, but the app was struggling to talk to RabbitMQ.

## Step 3: Was RabbitMQ actually running?
Turned out RabbitMQ was **disabled** on `rmq01`. Enabled and started it:
```
sudo systemctl enable rabbitmq-server
sudo systemctl start rabbitmq-server
sudo systemctl restart tomcat   # restart the app too — it doesn't reconnect cleanly on its own
```

## Step 4: The errors changed, but didn't go away
Same "Failed to check/redeclare auto-delete queue(s)" message kept appearing. This looked like a RabbitMQ permissions issue, so I checked:
```
sudo rabbitmqctl list_users
sudo rabbitmqctl list_permissions -p /
```
Both the `test` user and permissions were actually configured correctly. Not the cause.

## Step 5: The real test — is the app actually broken?
Instead of chasing the RabbitMQ logs further, I tested the app directly on its own server:
```
curl -I http://localhost:8080/
```
```
HTTP/1.1 200
```
**The app was working fine.** The RabbitMQ errors were just noisy background warnings — not what was causing the 502. This was the key pivot: stop debugging the app, start debugging the proxy.

## Step 6: Check Nginx's error log
```
sudo tail -n 30 /var/log/nginx/error.log
```
```
connect() failed (113: Unknown error) while connecting to upstream, ... upstream: "http://192.168.56.12:8080/"
```
Error `113` = **"No route to host"**. Nginx (`web01`) simply couldn't reach `app01` on port 8080 over the network — even though the app worked fine locally.

## Step 7: Root cause — firewalld
```
sudo systemctl status firewalld
```
`firewalld` was active on `app01` and blocking inbound traffic on port 8080. Fixed it:
```
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

Reloaded the site — working immediately. ✅

## Key Takeaway (Part 1)
A 502 Bad Gateway just means *"the proxy is up, but it can't get a valid response from the backend."* The cause can be anywhere in the chain:
- App server down
- App failing to start/deploy (dependency errors)
- Network/firewall blocking the proxy from reaching the app
- Wrong upstream config in Nginx

**The fastest way to isolate it:** test the app *locally* on its own server first (`curl -I http://localhost:8080/`). If that works, the problem isn't the app — it's the network path between the proxy and the app. That one test saved a lot of time chasing a red herring (the RabbitMQ warnings) instead of the actual cause (firewalld).

---

# Part 2: "User Not Found" on Login (Same Project, Next Day)

Once the site loaded, a new issue showed up: logging in with the default vProfile credentials returned **"user not found"** — even though everything about the app was otherwise working.

## Step 1: Confirm the account actually exists
```sql
USE accounts;
SELECT * FROM user WHERE username='admin_vp';
```
The `admin_vp` row was right there in the database. So this wasn't a missing-data problem.

## Step 2: Check the app's DB connection config
```
cat /usr/local/tomcat/webapps/ROOT/WEB-INF/classes/application.properties | grep -i jdbc
```
```
jdbc.url=jdbc:mysql://db01:3306/accounts
jdbc.username=admin
jdbc.password=admin123
```
Config looked correct — pointing at the right host, database, and credentials.

## Step 3: Verify the DB user's grants
```sql
SHOW GRANTS FOR 'admin'@'%';
```
```
GRANT ALL PRIVILEGES ON `accounts`.* TO `admin`@`%`
```
Grants were correct too, and manually connecting with those exact credentials from db01 worked fine. So on paper, everything checked out — yet the app still said "user not found."

## Step 4: Watch the logs live during the actual login attempt
This was the key move — instead of guessing, tail the logs *while* reproducing the error:
```
sudo journalctl -u tomcat -f
```
Then attempt the login in the browser. The real exception appeared immediately:
```
java.net.NoRouteToHostException: No route to host
    at com.mysql.cj.protocol.StandardSocketFactory.connect
```
**The app was never even reaching the database.** The generic "user not found" message was just the app's fallback error handling masking a connection failure — same pattern as the earlier RabbitMQ noise masking the real issue.

## Step 5: Same root cause, different server
Sound familiar? It was the exact same problem as Part 1, just one hop further down the chain:
```
sudo systemctl status firewalld    # on db01
```
`firewalld` was active on `db01`, blocking inbound port 3306 from `app01`. Fixed it the same way:
```
sudo firewall-cmd --permanent --add-port=3306/tcp
sudo firewall-cmd --reload
sudo systemctl restart tomcat      # on app01, to refresh the connection pool
```

Logged in with `admin_vp` / `admin_vp` — worked immediately. ✅

## Key Takeaway (Part 2)
An error message like "user not found" doesn't always mean what it says — many apps swallow the real exception (a DB connection failure, in this case) and show a generic, misleading message instead. When a "logical" error doesn't make sense given what you've already verified (the account clearly exists, credentials clearly work), **watch the live logs while reproducing the issue** rather than trusting the surface-level error text.

And once you've found one firewalld-blocked port in a multi-VM setup, check for it again on every other hop — it's a very repeatable mistake across `web01 → app01 → db01/rmq01/mc01`.

---
*Part of my hands-on DevOps learning journey — [Decoding DevOps on Udemy](https://www.udemy.com/course/decodingdevops/)*
