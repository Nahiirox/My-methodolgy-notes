# DNS Records / Virtual Hosts — Local Resolution

App redirecting to a hostname you can't reach → add it to `/etc/hosts`:

```bash
sudo sh -c 'echo "<TARGET>  <DOMAIN>" >> /etc/hosts'
```

Same IP, different vhost = same site until you find subdomains that serve different content. If full recursive fuzz on the IP turned up nothing interesting, pivot to subdomain enum next.