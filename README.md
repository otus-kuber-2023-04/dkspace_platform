# dkspace_platform
dkspace Platform repository

===============
docker build --rm -t web ./
docker run -d -p 8000:8000 -t web
docker exec -it {containerID} /bin/bash


 docker build --rm -t dmikos4/web:testimg .
 
check 
docker images

Upload to DockerHub (first login in CLI : docker login ) 
docker push dmikos4/web:testimg
