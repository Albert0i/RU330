#
# Import and expose environment variables
#
cnf ?= .env
include $(cnf)
export $(shell sed 's/=.*//' $(cnf))

#
# Main
#
.PHONY: help build up down ps logs 

help:
	@echo
	@echo "Usage: make TARGET"
	@echo
	@echo "Redis Replication Dockerize project automation helper for Linux version 1.0 (RU330)"
	@echo
	@echo "Targets:"
	@echo "	build		build custom image"
	@echo "	up  		start the replication"
	@echo "	down 		stop the replication"
	@echo "	ps 		show running containers"
	@echo "	logs		replication logs"

	@echo 
	@echo "	bash 		docker-compose exec primary bash"
	@echo "	cli 		docker-compose exec primary redis-cli"
	@echo "	sentinel	docker-compose exec sentinel1 redis-cli"
	@echo "	info		docker-compose exec primary redis-cli info replication"
	@echo "	infos		docker-compose exec sentinel1 redis-cli SENTINEL get-master-addr-by-name"
	@echo "	config		edit configuration"

#
# build custom image
#
build:
	docker-compose build

#
# start the replication
#
up:
	docker-compose up -d --remove-orphans
	@echo "Next, point your browser to http://localhost:5540"

#
# stop the replication
#
down:
	docker-compose down -v

#
# show running containers 
#
ps:
	docker-compose ps

#
# replication logs
#
logs:
	docker-compose logs 

#
# start bash on primary
#
bash:
	docker-compose exec primary bash

#
# start redis-cli on primary
#
cli:
	docker-compose exec primary redis-cli --pass ${STRONG_PASSWORD} --tls --cacert /usr/local/share/ca-certificates/ca.crt 

#
# start redis-cli on sentinel1
#
sentinel:
	docker-compose exec sentinel1 redis-cli

#
# replication info
#
info:	
	docker-compose exec primary redis-cli --pass ${STRONG_PASSWORD} --tls --cacert /usr/local/share/ca-certificates/ca.crt info replication

#
# replication info
#
infos:	
	docker-compose exec sentinel1 redis-cli SENTINEL get-master-addr-by-name myprimary

#
# edit configuration
#
config:
	nano .env

#
# Reference 
#
# 1. Multi-line bash commands in makefile
#    https://stackoverflow.com/questions/10121182/multi-line-bash-commands-in-makefile
# 2  Suppress echo of command invocation in makefile?
#    https://stackoverflow.com/questions/9967105/suppress-echo-of-command-invocation-in-makefile
#

#
# EOF (2024/06/08)
#

