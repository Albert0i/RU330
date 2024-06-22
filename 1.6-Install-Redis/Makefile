#
# Import and expose environment variables
#
cnf ?= .env
include $(cnf)
export $(shell sed 's/=.*//' $(cnf))

#
# Main
#
.PHONY: help prune config my-ciapp

help:
	@echo
	@echo "Usage: make TARGET"
	@echo
	@echo "Redis Dockerize project automation helper for Linux version 1.0 (RU330)"
	@echo
	@echo "Targets:"
	@echo "	build		build custom image"
	@echo "	up  		start the server"
	@echo "	down 		stop the server"	
	@echo "	cmd		run server cmd"
	@echo "	ps 		show running containers"
	@echo "	logs		server logs"
	@echo "	config		edit configuration"

#
# build custom image
#
build:
	docker-compose build

#
# start the server
#
up:
	docker-compose up -d --remove-orphans
	@echo "Next, point your browser to http://localhost:5540"

#
# stop the server
#
down:
	docker-compose down -v

#
# run server cmd
#
cmd:
	docker-compose exec redis cmd

#
# show running containers 
#
ps:
	docker-compose ps

#
# server logs
#
logs:
	docker-compose logs

#
# edit configuration
#
config:
	nano .env

#
# EOF (2024/06/22)
#