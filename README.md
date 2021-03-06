
# LA Nayarit Furniture
**An experimental Docker-Compose-Django-React Project.** The goal of this README is to document the development and deployment steps taken along the way to prevent future headaches repeating the same mistakes.

Services used:
1. **Django**: Our backend. We give complete control to Django for our admin console. Also handles our API calls using Django REST Framework.
2. **Postgres**: Our database. Interacts with our backend to read from and write to our data.
3. **React**: Our frontend. Everything the end-user experiences. Makes API calls to our backend to fetch data from our database.
4. **NGINX**: Our web server that also sets up or reverse proxy to the backend.
5. **GUNICORN**: Our web server utilizes gunicorn to allow the backend to communicate with NGINX.

This is not all there is to it, each service may require its own libraries and dependencies, but that's the big picture.
## Docker-Compose
For our single-host project, we will be looking at docker-compose to containerize our various services. 
### How does docker-compose allow communication between containers?
Take a look at the docker-compose.yml file. In our current setup, we have three 'services', or containers:
1. `database` : Containerizes Postgres
2. `backend` : Containerizes Django
3. `frontend` : Containerizes React

To make a new container, you must specify a hostname. This hostname can then be used across all our other containers. For example, NGINX uses the hostname 'http://backend:8000' to reverse-proxy into our backend. Django also uses the hostname 'database' to connect with our Postgres database within its settings.
### Volumes
A volume can be considered as a directory for persistent data across containers. This means that all data generated by both the source project directory and its containers will be shared. We use shared volumes for our static_root/media folders, shared between Django and React. That way, when we collectstatic within our backend container, these files are immediately available for reading and writing in the frontend folder and vice-versa. This is critical for serving files through NGINX.
## Django|Postgres
Django defaults to SQLite for debugging purposes, however, we will be using Postgres in production. Therefore, it is important to understand the relationship between Django and Postgres in order to make changes into our database when needed.
### Making Migrations
Say you've made changes to your models while in development. You upload to the server and are preparing for production. How do you apply these changes to your database? We need to run `manage.py makemigrations` to apply these changes into our database, which is easier said than done. In development, we'd be using SQLite; therefore, it is as easy as activating the virtual environment and executing the command. However, in production, this is different.
### Migrations in Production
Django cannot make migrations to a database it does not have access to. You will see that we use two different databases for development and product. If you attempt `makemigrations` anywhere outside of docker-compose's containers when `DEBUG=False`, then Django will yield an error. This is a common error that appears when building with docker-compose; we are running `manage.py makemigrations` at build time from its Dockerfile. In this error case, Django attempts to search for the Postgres database under the 'database' host, which does not exist at build time, only in run time. To fix this, we must set `DEBUG=False` to `DEBUG=True` before building, and then switch back after the build is complete.
### Accessing docker-compose Containers
After our services are up and running, we must apply our migrations from within our backend container. To create a shell inside our container, we run `docker exec -t -i <Container> /bin/sh`. These containers act as environments, so you would run commands as if you were inside the python virtual environment in development mode. For creating a shell in the backend, you access the backend container. From here, you can execute `makemigrations`,  `migrate`, `createsuperuser`, etc.
### Receiving API Calls
When it comes to making and receiving API calls, the architectural goal is to limit the amount of work the frontend must do to retrieve information from the backend. The result of this design is creating views using the Django REST Framework that contain all information the frontend needs from a single call. For example, if the frontend is to make an API call to retrieve all the data of a BlogPost model, then all fields of related models must be included in this view as well. This way, making another call to lookup related model fields, such as BlogCategory or BlogTag, would not be necessary.

## Django|React
## NGINX
The most important aspect of deployment, and quite possibly the largest source of deployment headaches. NGINX is our front facing application that redirects http and https requests to where they are suppose to go, according to our setup. If our static files, and/or media files are not showing up, NGINX is most likely the culprit.
### Gateway to our Backend
Django comes with an incredibly useful tool out of the box: the admin console. This prevents the need to create an admin console from scratch in React, saving us many, many hours of work. Since our frontend, by default, will handle all our views, we must make an exception to allow Django to handle requests made to access the admin console. To bridge these requests, NGINX will reverse-proxy all URI requests starting with /admin to our backend. Congratulations, we can now access our admin panel! We repeat this process for other views that we want Django to handle. In our case, our API handled by Django REST Framework, followed by Summernote. 
### NGINX vs Gunicorn
Django cannot serve our static or media files in production. According to Django's documentation, doing so is grossly inefficient. This is why we'll be using NGINX to serve our static and media files. So then, what is the purpose of Gunicorn? Gunicorn is a WSGI application that actually runs our backend server for production. The order of traffic goes:
1. Server receives an http request.
2. NGINX sits in front, deciding what to do with that request.
3. If it is a request to access the backend, then reverse proxy to Gunicorn.
4. Gunicorn determines what do do with the request, talking to our Django project.
### Aliasing
Now that know that NGINX serves our static/media files, we must tell it where they are located. Unlike the build folder of React, our static/media files are not located in our `usr` folder of the frontend container. Instead, we told docker to form our shared volumes on the root of our frontend container. Normally, this configuration is set by the `root` setting of our `nginx.conf`, so we must let NGINX know otherwise. So all there is to do is tell NGINX where it out frontend container are our volumes located.
## Common Commands
It's easy to forget which commands you used to perform basic tasks. Here you will find common commands used to perform said tasks.
### General
1. Firewall status:
`sudo ufw status`
1. Create firewall rule on port 8000:
`sudo ufw allow 8000`
1. Delete firewall rule on port 8000:
`sudo ufw delete allow 8000`
1. View processes:
`sudo netstat -tulpn`
1. Kill process:
`kill <processID>`
### docker-compose
1. Build docker-compose project:
`docker-compose run build`
1. Run docker-compose project:
`docker-compose up`
1. List docker containers:
`docker container ls`
1. Access docker container virtual environments:
`docker exec -t -i <VirtualEnv> /bin/sh`
1. Clean docker system files (for freeing space):
`docker system prune`
1. Restart docker:
`systemctl restart docker`
### Django
1. Setup migrations:
`manage.py makemigrations`
1. Apply migrations:
`manage.py migrate`
1. Create our static files:
`manage.py collectstatic`
### React
1. Install library:
`npm install <library>`
1. Build project:
`npm run build`
1. Start test server:
`npm run start`
