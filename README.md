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
  
## Installation on vm server

     sudo apt install haproxy
     sudo systemctl reload haproxy 

## HA Proxy configuration file location
  
    All settings are defined in the file **/etc/haproxy/haproxy.cfg** (or /etc/hapee-{version}/hapee-lb.cfg for HAProxy Enterprise). If you are using Docker, then this file is mounted as a volume into the container at the path **/usr/local/etc/haproxy/haproxy.cfg.**

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


    
