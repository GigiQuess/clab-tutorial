## deploy a lab
```
sudo containerlab deploy
```

## connecting to the nodes
```
# access CLI
docker exec -it clab-srlceos01-srl sr_cli
# access bash
docker exec -it clab-srlceos01-srl bash
```

## destroy a lab
```
sudo containerlab destroy
```

## edgeshark container up
```
curl -sL \
https://github.com/siemens/edgeshark/raw/main/deployments/wget/docker-compose.yaml \
| DOCKER_DEFAULT_PLATFORM= docker compose -f - up -d
```
- http://localhost:5001

## edgeshark container down
```
curl -sL \
https://github.com/siemens/edgeshark/raw/main/deployments/wget/docker-compose.yaml \
| DOCKER_DEFAULT_PLATFORM= docker compose -f - down
```
