# Cassandra Swarm
Dockerfile for creating a Cassandra cluster with Docker swarm, and be able to deploy it as one service.
It uses the DNS in Docker swarm to find all the other nodes and automatically sets them as seeds, and also sets the broadcast address to be the overlay network's ip address.



