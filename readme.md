# Heroku Buildpack: NGINX w/ GeoIP2

Extended from [Heroku's official NGINX buildpack](https://github.com/heroku/heroku-buildpack-nginx).

This vendors NGINX w/ the [GeoIP2 module](https://github.com/leev/ngx_http_geoip2_module/) and necessary [libmaxminddb](https://github.com/maxmind/libmaxminddb) inside a dyno and connects NGINX to an app server via UNIX domain sockets.

## Setup

* Setup the webserver to listen to the socket at `/tmp/nginx.socket`. e.g.:

	```ruby
	# config/puma.rb
	bind 'unix:///tmp/nginx.socket'
	```

* Touch `/tmp/app-initialized` when the process is ready to accept traffic. e.g.:

	```ruby
	# config/puma.rb
	on_worker_fork do
      FileUtils.touch('/tmp/app-initialized')
	end
	```


* Start the web server with a shell command through nginx-start. e.g.:

  ```ruby
  # Procfile
  web: bin/start-nginx bundle exec puma -C config/puma.rb
  ```

* Configure GeoIP2

	This is designed to work out of the box with the [MaxMind Buildpack](https://github.com/mantisadnetwork/heroku-buildpack-maxmind)

	**Note:** You must sign up for a [MaxMind Account](https://www.maxmind.com/en/geolite2/signup) and create a LicenceKey.

	Add the MaxMind Buildpack and set
	* `MAXMIND_KEY` your LicenceKey
	* `MAXMIND_EDITIONS` to `"GeoLite2-City, GeoLite2-ASN"` for the provided [`nginx.conf.erb`](config/nginx.conf.erb)

### Logging

NGINX will output the following style of logs:

```
[nginx] request_time=0.007 request_id=e2c79e86b3260b9c703756ec93f8a66d location=Austin, TX
```

### Customizable NGINX Config

You can provide your own NGINX config by creating `config/nginx.conf.erb` in your app. Start by copying the buildpack's [default config file](config/nginx.conf.erb).

### Opinionated Profided Config

* `geoip2_proxy 10.0.0.0/8;` is used because Heroku acts as a reverse proxy and the incoming IP address (the nginx `$remote_addr` variable) will be a private IP. This tells `geoip` to use the `X-Forwarded-For` header value which is the user's real IP.

* `geoip2 /app/GeoLite2-City.mmdb`
  * `$geoip2_city city names en;` This sets `$geoip2_city` to the city's English name if available (see `map` directive).
  * `$geoip2_state subdivisions 0 iso_code;` This sets `$geoip2_state` to the location's subdivisions ISO code. E.g. For states in the USA this would be TX for Texas.

* `geoip2 /app/GeoLite2-ASN.mmdb` For private networks w/o location
  * `$geoip2_asn autonomous_system_organization;` This sets `$geoip2_asn` to the name of the organization which owns the autonomous system. E.g. This could be `Facebook` if they're crawling your site. They will not have a city location.

* `map $geoip2_city $location` This normalizes the `$location` for logging
  * If `$geoip2_city` is blank that's a generally good sign the visiting IP is an ASN so `$location` is set to `$geoip2_asn`.
	* Otherwise `$location` is set to `"$geoip2_city, $geoip2_state"` e.g. `"Austin, TX"`.