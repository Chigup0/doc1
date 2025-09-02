# Local Setup Guide

This guide explains how to set up both the **Frontend (Next.js)** and **Backend (FastAPI)** locally for development. It also covers the environment variable configuration necessary for both parts of the application.

---

## Prerequisites

Before running the project locally, ensure the following prerequisites are installed.

### **System Requirements**

- **Python 3.10+**
- **Node.js 18+** (for Next.js frontend)
- **npm** or **yarn**
- **Git**
- **PostgreSQL** (or any other supported database for the backend)
- **Neo4j** (for graph-based AI data storage, optional but recommended)
- **AWS CLI** (for integration with AWS services)

---

## Backend Setup (FastAPI)

### Step 1: Clone the Repository

Clone the project repository to your local machine:

```bash
git clone <your-repository-url>
cd <project-directory>
```

### Step 2: Installing requirements

Move into ```backend``` folder and install requirements by: 
```
cd backend
pip install -r requirements.txt
```
### Step 3: Configure Environment variables both for frontend and backend

Create a ``` .env ``` file in backend like this example:
```
envexample
```
Then move to frontend and create a ``` .env.local ``` with contents like this :
``` code```

- Install dependencies
```
cd <project_name>
npm install
```
- Build and run the project
```
npm start
```
  Navigate to `http://localhost:8001`

- API Document endpoints

  swagger Spec Endpoint : http://localhost:8001/api-docs 

  swagger-ui  Endpoint : http://localhost:8001/docs 

The main purpose of this repository is to show a project setup and workflow for writing microservice. The Rest APIs will be using the Swagger (OpenAPI) Specification.




## Getting TypeScript
Add Typescript to project `npm`.
```
npm install -D typescript
```

## Project Structure
The folder structure of this app is explained below:

| Name | Description |
| ------------------------ | --------------------------------------------------------------------------------------------- |
| **dist**                 | Contains the distributable (or output) from your TypeScript build.  |
| **node_modules**         | Contains all  npm dependencies                                                            |
| **src**                  | Contains  source code that will be compiled to the dist dir                               |
| **configuration**        | Application configuration including environment-specific configs 
| **src/controllers**      | Controllers define functions to serve various express routes. 
| **src/lib**              | Common libraries to be used across your app.  
| **src/middlewares**      | Express middlewares which process the incoming requests before handling them down to the routes
| **src/routes**           | Contain all express routes, separated by module/area of application                       
| **src/models**           | Models define schemas that will be used in storing and retrieving data from Application database  |
| **src/monitoring**      | Prometheus metrics |
| **src**/index.ts         | Entry point to express app                                                               |
| package.json             | Contains npm dependencies as well as [build scripts](#what-if-a-library-isnt-on-definitelytyped)   | tsconfig.json            | Config settings for compiling source code only written in TypeScript    
| tslint.json              | Config settings for TSLint code style checking                                                |

## Building the project
### Configuring TypeScript compilation
```json
{
    "compilerOptions": {
      "target": "es5",
      "module": "commonjs",
      "outDir": "dist",
      "sourceMap": true
    },
    
    "include": [
      "src/**/*.ts"
      

    ],
    "exclude": [
      "src/**/*.spec.ts",
      "test",
      "node_modules"
    
    ]
  }

```
### Running the build
All the different build steps are orchestrated via [npm scripts](https://docs.npmjs.com/misc/scripts).
Npm scripts basically allow us to call (and chain) terminal commands via npm.

| Npm Script | Description |
| ------------------------- | ------------------------------------------------------------------------------------------------- |
| `start`                   | Runs full build and runs node on dist/index.js. Can be invoked with `npm start`                  |
| `build:copy`                   | copy the *.yaml file to dist/ folder      |
| `build:live`                   | Full build. Runs ALL build tasks       |
| `build:dev`                   | Full build. Runs ALL build tasks with all watch tasks        |
| `dev`                   | Runs full build before starting all watch tasks. Can be invoked with `npm dev`                                         |
| `test`                    | Runs build and run tests using mocha        |
| `lint`                    | Runs TSLint on project files       |
