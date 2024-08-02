# secure-by-design-conf-2024



#### WINDOW 1
clear
WORK_DIR_ROOT="/tmp"
rm -rf "${WORK_DIR_ROOT}/openziti-sidecar"

WORK_DIR="${WORK_DIR_ROOT}/openziti-sidecar"
mkdir -p "${WORK_DIR}"
cd $WORK_DIR
docker network create ziti-blue 
docker network create ziti-red 
docker run \
  --rm \
  --name ziti.controller \
  --network ziti-blue \
  --network ziti-red \
  --hostname controller.ziti \
  -p1280:1280 \
  -p3022:3022 \
  openziti/ziti-cli:1.1.8 edge quickstart



#### WINDOW 2
clear
WORK_DIR_ROOT="/tmp"
WORK_DIR="${WORK_DIR_ROOT}/openziti-sidecar"

ziti edge login localhost:1280 -u admin -p admin -y
ziti edge delete edge-router "blue-router"
ziti edge create edge-router "blue-router" --jwt-output-file="${WORK_DIR_ROOT}/blue-router.jwt" --tunneler-enabled
docker run \
  --rm -it \
  --network ziti-blue \
  --name ziti.blue-router \
  --dns 127.0.0.1 \
  --dns 1.1.1.1 \
  --user root \
  --cap-add NET_ADMIN \
  -e ZITI_CTRL_ADVERTISED_ADDRESS="controller.ziti" \
  -e ZITI_CTRL_ADVERTISED_PORT="1280" \
  -e ZITI_ENROLL_TOKEN="$(<"${WORK_DIR_ROOT}/blue-router.jwt")" \
  -e ZITI_BOOTSTRAP_CONFIG_ARGS="--private" \
  -e ZITI_ROUTER_MODE="tproxy" \
  -e ZITI_BOOTSTRAP=true \
  -e ZITI_ROUTER_ADVERTISED_ADDRESS="not.needed" \
  openziti/ziti-router:main


#### Window 3
clear
WORK_DIR_ROOT="/tmp"
WORK_DIR="${WORK_DIR_ROOT}/openziti-sidecar"

ziti edge login localhost:1280 -u admin -p admin -y
ziti edge delete edge-router "red-router"
ziti edge create edge-router "red-router" --jwt-output-file="${WORK_DIR_ROOT}/red-router.jwt" --tunneler-enabled
docker run \
  --rm -it \
  --network ziti-red \
  --name ziti.red-router \
  --dns 127.0.0.1 \
  --dns 1.1.1.1 \
  --user root \
  --cap-add NET_ADMIN \
  -e ZITI_CTRL_ADVERTISED_ADDRESS="controller.ziti" \
  -e ZITI_CTRL_ADVERTISED_PORT="1280" \
  -e ZITI_ENROLL_TOKEN="$(<"${WORK_DIR_ROOT}/red-router.jwt")" \
  -e ZITI_BOOTSTRAP_CONFIG_ARGS="--private" \
  -e ZITI_ROUTER_MODE="tproxy" \
  -e ZITI_ROUTER_ADVERTISED_ADDRESS="not.needed" \
  openziti/ziti-router:main


#### Window 4
clear
docker run --rm --name ziti.blue-hello --network ziti-blue --hostname blue.hello.ziti openziti/hello-world


#### Window 5
clear
ziti edge create config "blue-hello-intercept.v1" intercept.v1 \
  '{"portRanges":[{"high":80,"low":80}],"addresses":["hello.blue.ziti"],"protocols":["tcp"]}'
ziti edge create config "blue-hello-host.v1" host.v1 \
  '{"address":"blue.hello.ziti","port":8000,"protocol":"tcp"}'
ziti edge create \
	service "blue-hello" \
	--configs "blue-hello-intercept.v1,blue-hello-host.v1" \
	--role-attributes "blue-services"

ziti edge create service-policy "blue-hello.dial" Dial \
  --semantic AnyOf \
  --service-roles '#blue-services' \
  --identity-roles '#red-identities'
ziti edge create service-policy "blue-hello.bind" Bind \
  --semantic AnyOf \
  --service-roles '#blue-services' \
  --identity-roles '#blue-identities'

ziti edge update identity blue-router --role-attributes "blue-identities"
ziti edge update identity red-router --role-attributes "red-identities"

ziti edge policy-advisor identities -q


ziti edge list terminators

docker run -it --rm --name ziti.blue-client-on-red-network --network ziti-red openziti/quickstart curl hello.blue.ziti -m1
docker run -it --rm --name ziti.blue-client-on-red-network --network container:ziti.red-router openziti/quickstart curl http://hello.blue.ziti -m1





docker run -it --rm --name ziti.blue-client-on-red-network --network container:ziti.red-router openziti/quickstart curl hello.blue.ziti -m1
docker run -it --rm --name ziti.blue-client-on-red-network --network container:ziti.red-router curlimages/curl curl http://hello.blue.ziti -m1
docker run   --rm -it   --network container:ziti.red-router   --entrypoint bash   openziti/quickstart -c "curl http://hello.blue.ziti"


