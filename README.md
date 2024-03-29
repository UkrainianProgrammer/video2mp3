# video2mp3
App that converts video to an MP3 file and stores it in a database.

## How to use

Prerequisites! Need to install the following on your host:

* Docker
* Kubernetes
* MongoDB
* MySQL DB
* Email: turn on Less secure app access in the security section of your email
* Update /etc/hosts to include the following:
```
127.0.0.1 mp3converter.com
127.0.0.1 rabbitmq-manager.com
```
This will allow connection to the URLs which will be routed via localhost.

If you are using docker for Mac (like me), install Nginx using the following [guide](https://kubernetes.github.io/ingress-nginx/deploy/#docker-for-mac). This will install a k8s ingress agent to allow connection to the URLs mentioned above.

Note: to login to rabbitMQ GUI, use the following credentials:
```
username: guest
password: guest
```

This is a microservice application. Therefore, it needs to be deployed in several parts.

Each of the following applications contain a ```Dockerfile``` which needs to be run inside respective directory like so ```docker build .```

Afterwards, applications need to be tagged using ```docker tag <docker image id> <your repository name>:<version tag>```. 

Once images are tagged, push them to your repository ```docker push <image name>:<image tag>.```

Finally for each application, navigate to ```manifests``` folder and apply all YAML files using ```kubectl```: ```kubectl apply -f ./```

**To login to a server:**
curl POST http://mp3converter.com/login -u 'email':'password'

**To upload a video:**
curl -X POST -F 'file=@./<video_file>' -H 'Authorization: Bearer <token>' http://mp3converter.com/upload

**To download an MP3 file:**
curl --output <mp3_file>.mp3 -X GET  -H 'Authorization: Bearer <token>' "http://mp3converter.com/download?fid=<fid>"

Testing with Postman API is also an option.

### Testing

Login method verified via Postman with BasicAuth credentials:

![Login to Gateway](design/testing/Login_Gateway.png)

Example test result of uploading a test MP4 video:

```
curl -X POST -F 'file=@./test_video.mp4' -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Imluc3RlbjQ5MEBnbWFpbC5jb20iLCJleHAiOjE3MDUwMzAwODAsImlhdCI6MTcwNDk0MzY4MCwiYWRtaW4iOnRydWV9.4n7KMAuFnPVdQYCK4cVr3zcurmtrQ9s2Fx3V57ZnYX8' http://mp3converter.com/upload

success!%
```
Converter logs:
![Converter_upload_log](design/testing/Converter_upload_log.png)

Example test result of downloading the video:
```
curl --output mp3_download.mp3 -X GET  -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Im9sZWtzQGFnZ2llbmV0d29yay5jb20iLCJleHAiOjE3MDUxNzg3ODAsImlhdCI6MTcwNTA5MjM4MCwiYWRtaW4iOnRydWV9.gOeV5MAEE0eWeYDRWZOWohgcA3ut6cHCZmyDlKUG3wM' "http://mp3converter.com/download?fid=65a1a8292acb9733e31e282f"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  422k    0  422k    0     0   604k      0 --:--:-- --:--:-- --:--:--  621k
(venv) osofishchenko@Oleksandrs-MBP video2mp3 % ls
README.md               design                  mp3_download.mp3        src                     test.mp3                test_video.mp4
```

Example of an email received after video was uploaded to our Flask server:

![Email_example](design/testing/email_example.png)


## System design

Below is a diagram depcicting complete microservice-based architecture which will process video upload requests and will allow to download MP3 files.

![system design](design/video2mp3.png)

### How it works

There are eight components in this system:

* Client
* API Gateway
* Notification service
* Auth Service
* Video to MP3 service
* Storage DB
* Auth DB
* Message Queue

When a user uploads a video to be converted to an MP3 file, the request goes to API Gateway. The gateway sends the video file to the Video database (MongoDB). The Gateway also puts a message in a queue (RabbitMQ) to notify the downstream service that there is a video to be converted. The Video to MP3 service will consume the message from the queue. It will then get the ID of the video from the message, pull the video from the DB, convert the video to MP3, and store the MP3 file in the DB. The sevice then puts a new message in the queue to be consumed by the notification service to let the user know that the conversion job has finished. After reading the message, the notification service sends an email to the client with an MP3 ID. Client then uses the ID in combination with their JWT token to submit a request to API Gateway to download the file. Finally, the API Gateway will pull the file from MongoDB database and serve it to the client.
