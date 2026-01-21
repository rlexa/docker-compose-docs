# What is nginx auto proxy?!

[TODO](https://github.com/evertramos/nginx-proxy-automation)

# What is a "restarter"?!

```yaml
restarter:
  image: docker:cli
  restart: unless-stopped
  volumes: ["/var/run/docker.sock:/var/run/docker.sock"]
  entrypoint: ["/bin/sh", "-c"]
  command:
    - |
      while true; do
        current_epoch=$$(date +%s)
        target_epoch=$$(( $$(date -d "05:00" +%s) + 86400 ))
        sleep_seconds=$$(( target_epoch - current_epoch ))
        echo "$$(date) + $$sleep_seconds seconds"
        sleep $$sleep_seconds
        
        echo "restarting service" 
        docker restart <service-name-you-need-to-restart>
        echo "restarted"
      done
```
