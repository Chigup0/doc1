# Local Setup Guide

This guide explains how to set up both the **Frontend (Next.js)** and **Backend (FastAPI)** locally for development. It also covers the environment variable configuration necessary for both parts of the application.




### Local run using docker

TO run the system using docker, just run the command

docker-compose build
docker-compose up -d

This exposes the frontend to the 3000 port, env variables need to set by the user

## Running The scripts

``cd backend/scripts``

```python populate_users.py <path of target folder>```

often time it was observed that tee system was unable to load .env from backend 

hence 

add the following code to top of populate_users.py

``load_dotenv(dotenv_path=Path(__file__).parent.parent / '.env') ``