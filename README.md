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

