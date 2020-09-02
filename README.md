## Kubernetes Cluster with highly available Mediawiki Site

This solution deploys a Mediawiki, with MariaDB. 

I have designed for scalability out of the box by using memcached for mediawiki and putting the entire website behind haproxy.

# Usage

You can deploy it with:

    juju deploy wiki-scalable

After deployment you need to expose the Mediawiki service

    juju expose loadbalancer

Then run a `juju status loadbalancer` to get the public address. The browse to that address in your browser to configure and use the wiki.

# For scaling out

You can add and remove more mediawiki instances to horizontally scale based on your convenience:

    juju add-unit wiki
