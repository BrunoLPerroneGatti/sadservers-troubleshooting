## Problem

There's a "dockerized" Node.js web application in the /home/admin/app directory. Create a Docker container so you get a web app on port _:8888_ and can _curl_ to it.  
  
Right now curl localhost:8888 does not return the app response — something else may already be answering on that port. For the solution to be valid, there should be only one running Docker container (any image/container name is fine).  
  
NOTE: The base image _node:15.7-alpine_ declared in Dockerfile is present locally (docker images).  
  
NOTE: nginx service is not needed in this server.

## Solution

Attempt to access the app server locally:

```bash
curl localhost:8888
```
 
 This returns the default Nginx page, meaning that Nginx is occupying the required host port, preventing the Node.js container from using it.

Since Nginix is not needed in this server we can stop it:

```bash
sudo systemctl stop nginx
```

Inspect existing containers and images:

```bash
docker ps -a
docker images
```

Output shows a stopped container named `app` that had exited with status `1`, along with the existing `app:latest` and `node:15.7-alpine` images.

Check the container logs

```bash
docker logs app
```

We get an important error:

```bash
Error: Cannot find module '/usr/src/app/serve.js'
code: 'MODULE_NOT_FOUND'
```

This indicated that Node.js was trying to execute a file that did not exist.

The directory contained `server.js`, but the Dockerfile specifies:

```
CMD [ "node", "serve.js" ]
```

Fix the Dockerfile with the appriate `server.js` file. 

The Dockerfile also contains ``EXPOSE 8880``. The application listens on port `8888` by default, as defined in `server.js`:

```bash
port = process.env.PORT || 8888
```

Therefore, updating the Dockerfile to `EXPOSE 8888` is good practice, although it is not strictly required for the explicit port mapping used here.
`EXPOSE` documents the container's intended port. It does not publish that port on the host. The `-p` option does that.

Rebuild the Docker image:

```bash
docker build -t app /home/admin/app
```

Remove the old stopped container:

```bash
docker rm -f app
```


Start the corrected container:

```bash
docker run -d --name app -p 8888:8888 app
```

Verify the solution

```bash
curl localhost:8888
# Expected output:
Hello World!
```