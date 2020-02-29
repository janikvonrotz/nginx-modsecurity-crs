
# Nginx ModSecurity CRS

Docker Compose project to setup a ModSecurity enabled Nginx container with the CRS.

## Usage

Clone this repository.

```bash
cd ~
git clone https://github.com/janikvonrotz/nginx-modsecurity-crs.git
```

Sync the submodules.

```bash
git submodule update --init
```

Run Docker Compose.

```bash
docker-compose up -d
```

### Audit

By default the Nginx container starts in audit mode. Before enabling the security engine you want to ensure that ModSecurity does not block any false positives. Therefore you evalute your application in audit mode.

Here is an example of the audit process:

Tail the audit log.

```bash
docker exec -it nginx-modsecurity-crs_waf_1 tail -f /var/log/modsec_audit.log
```

Trigger a security rule with curl.

```bash
curl -I 'https://localhost/?param="><script>alert(1);</script>' --insecure
```

The request has not been blocked and you should get a response like this:


```html
HTTP/1.1 200 OK
Server: nginx/1.15.12
Date: Wed, 26 Feb 2020 13:42:17 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 16 Apr 2019 13:08:19 GMT
Connection: keep-alive
ETag: "5cb5d3c3-264"
Accept-Ranges: bytes
```

The output of the audit log looks like this:

```txt
ModSecurity: Warning. detected XSS using libinjection. 
[file "/etc/modsecurity/crs/rules/REQUEST-941-APPLICATION-ATTACK-XSS.conf"]
[line "37"] [id "941100"] [rev ""] [msg "XSS Attack Detected via libinjection"] 
[data "Matched Data: XSS data found within ARGS:param: "><script>alert(1);</script>"] 
[severity "2"] [ver "OWASP_CRS/3.2.0"] [maturity "0"] [accuracy "0"] [tag "application-multi"] 
[tag "language-multi"] [tag "platform-multi"] [tag "attack-xss"] [tag "OWASP_CRS"] 
[tag "OWASP_CRS/WEB_ATTACK/XSS"] [tag "WASCTC/WASC-8"] [tag "WASCTC/WASC-22"] 
[tag "OWASP_TOP_10/A3"] [tag "OWASP_AppSensor/IE1"] [tag "CAPEC-242"] [hostname "172.22.0.1"] 
[uri "/"] [unique_id "158272291834.052399"] 
[ref "v12,28t:utf8toUnicode,t:urlDecodeUni,t:htmlEntityDecode,t:jsDecode,t:cssDecode,t:removeNulls"]
```

An XSS attack has ben detected by rule number `941100`.  
Now you would decide wether to disable this rule by updating the `etc/modsecurity/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf` file or update your application.

### Production

If your application has been tested and the audit log does not have any new entries, the security engine can be enabled.

Edit the ModSecuity config to do so.

**etc/modsecurity.d/modsecurity.conf**

```
...
SecRuleEngine On
...
```

Restart the Nginx container.

```
docker-compose restart
```

Trigger the security rule.

```bash
curl -I 'https://localhost/?param="><script>alert(1);</script>' --insecure
```

And you should get a response like this:

```html
HTTP/1.1 403 Forbidden
Server: nginx/1.15.12
Date: Wed, 26 Feb 2020 13:35:17 GMT
Content-Type: text/html
Content-Length: 154
Connection: keep-alive
```

The request has been blocked.

## Templates

The `etc` folder of this repo contains various config files. These files have been copied either from the Nginx Docker image or the ModSecurity Core Rule Set repository.

Here is a list of the config files an their source:

[etc/modsecurity/crs-setup.conf](https://github.com/SpiderLabs/owasp-modsecurity-crs/blob/v3.3/dev/crs-setup.conf.example)  
[etc/modsecurity/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf](https://github.com/SpiderLabs/owasp-modsecurity-crs/blob/v3.3/dev/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf.example)  
[etc/modsecurity.d/modsecurity.conf](https://github.com/SpiderLabs/ModSecurity/blob/v3/master/modsecurity.conf-recommended)  
[etc/nginx/conf.d/default.template](https://github.com/CRS-support/modsecurity-docker/blob/v3/nginx-nginx/Dockerfile)  

### Edits

I wanna show you which templates I have edited in what way.

The default security rule has been enabled.

**etc/modsecurity/crs-setup.conf**

```txt
...
SecDefaultAction "phase:1,deny,log"
...
```

Rules are included by wildcard.

**etc/modsecurity.d/include.conf**

```
...
Include /etc/modsecurity/crs/rules/*.conf
...
```
