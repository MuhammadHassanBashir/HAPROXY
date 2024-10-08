# HA-PROXY

## Use

    Use for 
    1- loadbalancing 
    2- Security
    3- ACL
    4- Source based rate limiting
    5- DDoS prevention
    6- XSS (Cross site scripting) prevention
    7- CORS
    8- Security Headers implementation as per OWASP

## Domain point of vm

     1- First, create an HAProxy VM on the cloud and enable both HTTP and HTTPS ports on the VM.
     2- Next, obtain the VM's public IP and navigate to the project where your domain is hosted. In Google Cloud, go to Cloud DNS and point the domain to the server's IP, for example, set haproxy.disearch.ai to the VM IP address.
     3- Then, install the certificate on the VM using Let's Encrypt. Make sure the domain mapping is complete before this step, as Let's Encrypt will issue the certificate for the mapped domain, such as haproxy.disearch.ai. After this, Let's Encrypt will issue a certificate for your domain and send it to your VM. When traffic is sent using the domain from a browser, it will first go to the domain's DNS, then to your VM, and retrieve the certificate from the VM.
  
## now Install ha proxy on vm server

     sudo apt install haproxy
     sudo systemctl reload haproxy 

## HA Proxy configuration file location
  
    All settings are defined in the file **/etc/haproxy/haproxy.cfg** (or /etc/hapee-{version}/hapee-lb.cfg for HAProxy Enterprise). If you are using Docker, then this file is mounted as a volume into the container at the path **/usr/local/etc/haproxy/haproxy.cfg.**


## Tested HA Proxy .cfg file written by Areez
    
    global                                                                                          ----> haproxy global section
    	log /dev/log	local0
    	log /dev/log	local1 notice
    	chroot /var/lib/haproxy
    	stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    	stats timeout 30s
    	user haproxy
    	group haproxy
    	daemon
            
            # Load Lua CORS Library
            lua-load /etc/haproxy/cors.lua
    
    	# Default SSL material locations
    	ca-base /etc/ssl/certs
    	crt-base /etc/ssl/private
    
    	# See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
            ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
            ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
            ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets
    
    defaults                                                                                   ----> haproxy default section
    	log	global
    	mode	http
    	option	httplog
    	option	dontlognull
            timeout connect 5000
            timeout client  50000
            timeout server  50000
    	errorfile 400 /etc/haproxy/errors/400.http
    	errorfile 403 /etc/haproxy/errors/403.http
    	errorfile 408 /etc/haproxy/errors/408.http
    	errorfile 500 /etc/haproxy/errors/500.http
    	errorfile 502 /etc/haproxy/errors/502.http
    	errorfile 503 /etc/haproxy/errors/503.http
    	errorfile 504 /etc/haproxy/errors/504.http
    
    frontend https_front                                                                      ---> haproxy frontend section, giving fronted name here **https_front**       
        bind *:443 ssl crt /etc/ssl/private/haproxy.disearch.ai.pem alpn h2,http/1.1          ---->fronted is listen on all ip with 443 port, give ssl certificate path and enable http2 protocol
        mode http                                                                             -----> we have 2 mode http/tcp   use to open traffic packet header and get multiple information like URL, and use this information for traffic routing.
        option forwardfor                                                                     ---> This option is used to add the X-Forwarded-For header to HTTP requests. The X-Forwarded-For header is used to track the original client IP address when the request passes through a proxy (like HAProxy).
        http-request set-header X-Forwarded-Proto: https                                      ----> This directive sets the X-Forwarded-Proto header in the request. The X-Forwarded-Proto header is used to tell the backend server whether the original client used HTTP or HTTPS protocol when making the request.
        default_backend web_backend                                                           ----> referring fronted to backend
    
        # DDoS & Rate Limiting
        # Allows HAProxy to track statistics (like request rates, current connections, etc.) per IP address. This is useful for applying rate limits or connection limits.
        stick-table type ip size 1m expire 30s store http_req_rate(30s),conn_cur
    
        # ACL flags an IP as abuse if it sends more than 100 HTTP requests within 10 seconds, which is considered a potential DoS (Denial of Service) attack
        acl abuse src_http_req_rate(https_front) gt 100
    
        # ACL flags an IP if it has more than 10 concurrent connections, which might indicate an abuse of resources or a bot attempting to flood the server with connections.
        acl too_many_conns src_conn_cur gt 10
    
        # If the abuse ACL condition is met, the incoming HTTP request will be denied, effectively blocking further traffic from that IP.
        http-request deny if abuse
    
        # If the too_many_conns ACL condition is met, the TCP connection will be rejected, preventing the client from opening additional connections.
        tcp-request content reject if too_many_conns
        http-request track-sc0 src
        http-request deny deny_status 429 if { sc_http_req_rate(0) gt 20 }
    
    
    
        # Security Headers (OWASP)
        # To Disable iFrame embedding
        http-response set-header X-Frame-Options "DENY"
    
        # Useful for security because an attacker could upload a file with an incorrect file type (like a .jpg file with malicious JavaScript code inside)
        http-response set-header X-Content-Type-Options "nosniff"
    
        # It tells browsers that your site should only be accessed over HTTPS, and never HTTP (insecure)
        http-response set-header Strict-Transport-Security "max-age=31536000; includeSubDomains"
    
        # Your site will not send any referrer information when a user clicks on a link to navigate to another site
        http-response set-header Referrer-Policy "no-referrer"
    
        # Helps prevent cross-site scripting (XSS) and other code injection attacks
        http-response set-header Content-Security-Policy "default-src 'self'; script-src 'self';"
    
        # This header enables a built-in XSS (cross-site scripting) filter in the browser.
        http-response set-header X-XSS-Protection "1; mode=block"
    
    
    
        # CORS
    
        # ACL flags requests that use the OPTIONS method, which are typically preflight CORS requests.
        acl is_preflight_options method OPTIONS
    
        # Log the Origin header
        http-request capture req.hdr(Origin) len 20
    
        # CORS to only allow request from example.com & example.org
        acl allowed_host1 hdr(host) -i example.com
        acl allowed_host2 hdr(host) -i example.org
        http-request deny if !allowed_host1 !allowed_host2

    
    backend web_backend                                          --> haproxy backend section, giving backend name here **web_backend**
        mode http
        server local_server localhost:8000 check                  ------> testing nginx container is running locally on 8000 port 

## Save **haproxy.cfg** changes by reloading haproxy service

     sudo systemctl reload haproxy 

## Command to verify haproxy file is correct or not..

    sudo haproxy -c -f /etc/haproxy/haproxy.cfg

## Testing
   
    Create an NGINX container on 8000 port on the server where HAProxy is running. Then test the HAProxy configuration. A successful configuration will display the NGINX home page. You can also verify the working of  ddos , owasp , cors set in ha proxy, get help from chatgpt for this.

# Haproxy configuration for my scenerio 

    File path **/etc/haproxy/haproxy.cfg**

    global

    default

    frontend http_front
      bind *:443 ssl crt /etc/haproxy/certs/ alpn h2,http/1.1           -------> give here your SSL certificate path, and by using alpn h2,http/1.1 haproxy also support to http2 protocol
      mode http
      default backend http_back
    
      # DDoS & Rate Limiting                                          -------------->> DDos attack prevention section
      stick-table type ip size 1m expire 10s store http_req_rate(10s),conn_cur
      acl abuse src_http_req_rate(http_front) gt 100
      acl too_many_conns src_conn_cur gt 10
      http-request deny if abuse
      tcp-request content reject if too_many_conns
    
      # Security Headers (OWASP)
      http-response set-header X-Frame-Options "DENY"            -------------------->> iframe prevention section, you can verify this my saying chatgpy to create ifram againt your domain. Once run you will see the result           
      http-response set-header X-Content-Type-Options "nosniff"
      http-response set-header Strict-Transport-Security "max-age=31536000; includeSubDomains"
      http-response set-header Referrer-Policy "no-referrer"
      http-response set-header Content-Security-Policy "default-src 'self'; script-src 'self';"
      http-response set-header X-XSS-Protection "1; mode=block"
    
      # CORS
      acl is_preflight_options method OPTIONS
      http-request use-service lua.cors if is_preflight_options
      http-response set-header Access-Control-Allow-Origin "*"
      http-response set-header Access-Control-Allow-Methods "GET, POST, OPTIONS"
      http-response set-header Access-Control-Allow-Headers "Content-Type"
    
    backend http_back  
    mode http
    server blog1 localhost:8000 check
     
                            ---------------------> configuration for backend server                       
    listen


## HA Proxy features

Simple installation steps and basic testing..

    configure this file  **/etc/haproxy/haproxy.cfg** 

      defaults                      
          mode http
          timeout client 10s
          timeout connect 5s
          timeout server 10s 
          timeout http-request 10s
    
      frontend myfrontend
          bind 127.0.0.1:80    

        This example also includes a defaults section, which defines settings that are shared across all sections that follow. It sets timeouts for how long HAProxy should wait for a client to send data (timeout client), how long to wait when trying to connect to a backend server (timeout connect), how long to wait for the server to send back data (timeout server), and how long to wait for the client to send a complete HTTP request (timeout http-request). These are just good safety measures that will help avoid common problems.
    
    The mode sets how HAProxy treats requests. It can be either http or tcp, with the former used when working with HTTP web applications. In this mode, HAProxy is able to inspect metadata of an HTTP request, such as headers, cookies, and the URL path, when deciding how to route the request.

    
        sudo systemctl restart haproxy
        
        The load balancer is now listening on the localhost address, 127.0.0.1, at port 80. If you make a request using curl, you’ll get back a response:
        
        curl 127.0.0.1:80
        <html><body><h1>503 Service Unavailable</h1>
        No server is available to handle this request.
        </body></html>
       
        Granted, there’s no reply from a server because we haven’t configured any servers yet. Nevertheless, you can see that HAProxy is functional. The table below lists a few other ways that you could set the bind line:
        
        bind 0.0.0.0:80
        
        Listen on all IP addresses assigned to this server at port 80.
        
        bind :80
        
        The same as specifying 0.0.0.0 for the address.
        
        bind :80,:8080
        
        Listen on ports 80 and 8080. (Do not add a space between ports)
        
        bind :6379-6390


## Configure backend 

    
        defaults
          mode http
          timeout client 10s
          timeout connect 5s
          timeout server 10s 
          timeout http-request 10s
        
        frontend myfrontend
          bind 127.0.0.1:80
          default_backend myservers
        
        backend myservers
          server server1 127.0.0.1:8000

        now you will see the response from backend. Example like this...

        $ curl 127.0.0.1:80
        <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
        <html>
        ...
        </html>

## Loadbalancing

HP Proxy loadbalancing traffic to backend server. 

    configure this file  **/etc/haproxy/haproxy.cfg**  
             
    frontend myfrontend
      bind 127.0.0.1:80
      default_backend myservers
    
    backend myservers
      server server1 127.0.0.1:8000
      server server2 127.0.0.1:8001
      server server3 127.0.0.1:8002   

## ACL 

Set Rules for Edge Cases
    
    Suppose you wanted to send all requests that arrive on port 80 to servers 1 and 2, but if a request arrives at port 81, it should go to server 3. Consider the following configuration that achieves that:
        
    frontend myfrontend
      bind 127.0.0.1:80,127.0.0.1:81
      use_backend special if { dst_port 81 }
      default_backend myservers
    
    backend myservers
      server server1 127.0.0.1:8000
      server server2 127.0.0.1:8001
    
    backend special
      server server3 127.0.0.1:8002


I’ve separated those addresses with a comma. I could also have written a second bind line, like this:

    frontend myfrontend
      bind 127.0.0.1:80
      bind 127.0.0.1:81
      use_backend special if { dst_port 81 }
      default_backend myservers

The table below lists examples of other ACLs that you might use to route traffic to different backends. Before you can use some of them, you will need to set mode http in your frontend and backend sections, which allows HAProxy to evaluate HTTP data in a message.
    
    ACL
    
    Description
    
    if { path_beg /api/ }
    
    Route API requests.
    
    if { path_end .jpg .png }
    
    Route requests for images.
    
    if { hdr(host) -m dom example.local }
    
    Routes requests for the domain example.local.
    
    if { src 127.0.0.1/8 }
    
    Route requests originating from the given IP address range.
    
    if { method POST PUT }
    
    Route POST and PUT requests.
    
    if { url_param(region) europe }
    
    Route requests that have a URL parameter named region that is set to europe.

## Enabling SSL with HAProxy
    
    HAProxy version 1.5, which was released in 2016, introduced the ability to handle SSL encryption and decryption without any extra tools like Stunnel or Pound. Enable it by editing your HAProxy configuration file, adding the ssl and crt parameters to a bind line in a frontend section. Here’s an example:
        
        frontend www.mysite.com
            bind 10.0.0.3:80
            bind 10.0.0.3:443 ssl crt /etc/ssl/certs/mysite.pem
            default_backend web_servers
    
            The ssl parameter enables SSL termination for this listener. The crt parameter identifies the location of the PEM-formatted SSL certificate. This certificate should contain both the public certificate and the private key. That’s it for turning on this feature. Once traffic is decrypted it can be inspected and modified by HAProxy, such as to alter HTTP headers, route based on the URL path or Host, and read cookies. The messages are also passed to backend servers with the encryption stripped away.

        Although you lose some of the benefits of SSL termination by doing so, if you prefer to re-encrypt the data before relaying it, then you’d simply add an ssl parameter to your server lines in the backend section. Here’s an example:
        
        backend web_servers
            balance roundrobin
            server server1 10.0.1.3:443 check maxconn 20 ssl
            server server2 10.0.1.4:443 check maxconn 20 ssl
      
        When HAProxy negotiates the connection with the server, it will verify whether it trusts that server’s SSL certificate. If the server is using a certificate that was signed by a private certificate authority, you can either ignore the verification by adding verify none to the server line or you can store the CA certificate on the load balancer and reference it with the ca-file parameter. Here’s an example that references the CA PEM file:
        
        backend web_servers
            balance roundrobin
            server server1 10.0.1.3:443 check maxconn 20 ssl ca-file /etc/ssl/certs/ca.pem
            server server2 10.0.1.4:443 check maxconn 20 ssl ca-file /etc/ssl/certs/ca.pem
       
        If the certificate is self-signed, in which case it acts as its own CA, then you can reference it directly.

​
## Redirecting From HTTP to HTTPS

    When a person types your domain name into their address bar, more likely than not, they won’t include https://. So, they’ll be sent to the http:// version of your site. When you use HAProxy for SSL termination, you also get the ability to redirect any traffic that is received at HTTP port 80 to HTTPS port 443.
    
    Add an http-request redirect scheme line to route traffic from HTTP to HTTPS, like this:
    
    frontend www.mysite.com
        bind 10.0.0.3:80
        bind 10.0.0.3:443 ssl crt /etc/ssl/certs/mysite.pem
        http-request redirect scheme https unless { ssl_fc }
        default_backend web_servers
    
    This line uses the unless keyword to check the ssl_fc fetch method, which returns true unless the connection used SSL/TLS. If it wasn’t used, the request is redirected to the https scheme. Now, all traffic will end up using HTTPS.
    
    This paves the way to adding an HSTS header, which tells a person’s browser to use HTTPS from the start the next time they visit your site. You can add an HSTS header by following the steps described in our blog post, HAProxy and HTTP Strict Transport Security (HSTS) Header in HTTP Redirects.
        

A typical load balancer configuration file looks like the following:

    global
        # process settings
    defaults
        # default values for sections below
    frontend
        # server the clients connect to
    backend
        # servers for fulfilling client requests
    listen
        # complete proxy definition

## HTTP

    Although HAProxy can load balance HTTP requests in TCP mode, in which the connections are opaque and the HTTP messages are not inspected or altered, it can also operate in HTTP mode. In HTTP mode, the load balancer can inspect and modify the messages, and perform protocol-specific actions. To enable HTTP mode, set the directive mode http in your frontend and backend section.

    Below, we describe features related to distinct versions of the HTTP protocol.

    HTTP/2 
    You can load balance HTTP/2 over:
    
    encrypted HTTPS when OpenSSL 1.0.2 or newer is available on the server
    unencrypted HTTP (known as h2c)
    Most browsers support HTTP/2 over HTTPS only, but you may find it useful to enable h2c between backend services (e.g. gRPC services).

    HTTP/2 over HTTPS to the client 
    Available since
    HAProxy 1.8
    HAProxy Enterprise 1.8r1
    HAProxy ALOHA 10.0
    HTTP/2 is enabled by default between clients and load balancer in HAProxy ALOHA 15.5 / HAProxy Enterprise 2.8r1 and up. You do not need to specify the alpn extension, because it has a default value of h2,http/1.1 for HTTPS bind lines. Note that ALPN works only for HTTPS bind lines, and so HTTP/2 requires HTTPS. Clients that lack support for HTTP/2 will be automatically reverted to HTTP/1.1. The load balancer server must have OpenSSL 1.0.2 or newer.
    
    frontend www
      mode http
      bind :443 ssl crt /path/to/cert.crt
      default_backend servers
    
    For HAProxy ALOHA 15.0 / HAProxy Enterprise 2.7r1 and older, you will need to specify both the extension and protocols:

    frontend www
      mode http
      bind :443 ssl crt /path/to/cert.crt alpn h2,http/1.1          ------------> use this **alpn h2,http/1.1 ** to enable http2, so haproxy also support this.   
      default_backend servers

