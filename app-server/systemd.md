# Running server as systemd unit

## Configuring unit

Here you can find an example of systemd unit file that can be used to run RoadRunner as a daemon on
a server:

{% code title="rr.service" %}

```ini
[Unit]
Description=High-performance PHP application server
Documentation=https://docs.roadrunner.dev/

[Service]
ExecStart=/usr/local/bin/rr serve -c /var/www/.rr.yaml

Type=notify

Restart=always
RestartSec=30

[Install]
WantedBy=default.target 
```

{% endcode %}

Where:

- `/usr/local/bin/rr` - path to the RoadRunner binary file
- `/var/www/.rr.yaml` - path to the RoadRunner configuration file

{% hint style="warning" %}
These paths are just examples, and the actual paths may differ depending on the specific
server configuration and file locations. You should update these paths to match the actual paths used in your server
setup.
{% endhint %}

You should also update the `ExecStart` option with your own configuration and save the file with a suitable name,
such as `rr.service`. Usually, such user unit files are located in the `.config/systemd/user/` directory. To enable the
service, you should run the following commands:

{% code %}

```bash
systemctl enable --user rr.service
```

{% endcode %}

and

{% code %}

```bash
systemctl start rr.service
```

{% endcode %}

This will start RoadRunner as a daemon on the server.

For more information about systemd unit files, the user can refer to the
following [link](https://wiki.archlinux.org/index.php/systemd#Writing_unit_files).

## Status and logs

Make sure that the systemd service has started successfully, use the command:

{% code %}

```bash
systemctl status rr.service
```
{% endcode %}

Example output:

```
● rr.service - High-performance PHP application server
     Loaded: loaded (/lib/systemd/system/rr.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2024-03-12 19:20:55 UTC; 35min ago
     Docs: https://roadrunner.dev/docs
     CGroup: /system.slice/rr.service
             ├─2835793 /usr/local/bin/rr serve -c /var/www/.rr.yaml
             ├─2895807 /usr/local/bin/php worker.php
             ├─2897198 /usr/local/bin/php worker.php
             ├─2897765 /usr/local/bin/php worker.php
```

To view logs in real time, use the command:

{% code %}

```bash
journalctl -f -u rr.service
```

{% endcode %}

Example output with `info` [level logs](../lab/logger.md#level):

```bash
Mar 12 20:12:53 server systemd[1]: Starting PHP application server...
Mar 12 20:12:55 server rr[2936506]: [INFO] RoadRunner server started; version: 2023.3.10, buildtime: 2024-02-01T22:33:17+0000
Mar 12 20:12:55 server rr[2936506]: [INFO] sdnotify: notified
Mar 12 20:12:55 server systemd[1]: Started PHP application server.
```

## `sd_notify` protocol

Roadrunner supports [sd_notify](https://man.archlinux.org/man/sd_notify.3.en) protocol. You can use it to notify systemd about the readiness of your application. Don't forget to add `Type=notify` directive to the `Service` section, Roadrunner will automatically detect systemd and send the notification. The only one option which might be
configured is watchdog timeout. By default, it's turned off. You can enable it by setting the following option in your
`.rr.yaml` config:

{% code title=".rr.yaml" %}

```yaml
endure:
  log_level: error
  watchdog_sec: 60 # watchdog timeout in seconds
```

{% endcode %}
