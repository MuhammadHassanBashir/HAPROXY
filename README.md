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
                


    
