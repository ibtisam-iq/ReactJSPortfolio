# Dockerizing a ReactJS Project

The project is a frontend-only ReactJS application with no backend or database integration, making it a 1-tier application. This means it handles the presentation layer (UI/UX) only, without connecting to a backend or database.

## Project Structure

```
.
├── .gitignore
├── package.json
├── Portfolio.png
├── public
│   └── index.html
├── README.md
└── src
    ├── App.jsx
    ├── assets
    │   ├── avatar1.jpg
    │   ├── avatar2.jpg
    │   ├── avatar3.jpg
    │   ├── avatar4.jpg
    │   ├── bg-texture.png
    │   ├── cv.pdf
    │   ├── me-about.jpg
    │   ├── me.png
    │   ├── portfolio1.jpg
    │   ├── portfolio2.jpg
    │   ├── portfolio3.jpg
    │   ├── portfolio4.jpg
    │   ├── portfolio5.png
    │   └── portfolio6.jpg
    ├── components
    │   ├── about
    │   │   ├── about.css
    │   │   └── About.jsx
    │   ├── contact
    │   │   ├── contact.css
    │   │   └── Contact.jsx
    │   ├── experience
    │   │   ├── experience.css
    │   │   └── Experience.jsx
    │   ├── footer
    │   │   ├── footer.css
    │   │   └── Footer.jsx
    │   ├── header
    │   │   ├── CTA.jsx
    │   │   ├── header.css
    │   │   ├── Header.jsx
    │   │   └── HeaderSocials.jsx
    │   ├── nav
    │   │   ├── nav.css
    │   │   └── Nav.jsx
    │   ├── portfolio
    │   │   ├── portfolio.css
    │   │   └── Portfolio.jsx
    │   ├── services
    │   │   ├── services.css
    │   │   └── Services.jsx
    │   └── testimonials
    │       ├── testimonials.css
    │       └── Testimonials.jsx
    ├── index.css
    └── index.js
```

### Breakdown of the Structure
- `public/index.html`: The main entry point for the frontend.
- `src/`: Contains the React components that handle the user interface, including the various sections (about, contact, experience, portfolio, etc.).
- `package.json`: Manages dependencies and scripts for building and running the React app.

## Dockerizing the ReactJS Project

For ReactJS project, there are a few ways to write the Dockerfile. Here’s a guide on each approach, with pros and cons for each.

### 1. Single Build Dockerfile

A single build Dockerfile is a simple way to set up your container. It compiles your React app and serves it in one stage.

#### Dockerfile:
```Dockerfile
# Use official Node.js image
FROM node:18

# Set working directory
WORKDIR /app

# Copy package.json and install dependencies
COPY package.json ./
RUN npm install

# Copy all files
COPY . .

# Build the React app for production
RUN npm run build

# Install Nginx to serve the app
RUN apt-get update && apt-get install -y nginx

# Copy the build output to the Nginx directory
RUN cp -r /app/build/* /var/www/html/

# Expose port 80
EXPOSE 80

# Start Nginx to serve the app
CMD ["nginx", "-g", "daemon off;"]
```

#### Key Points:
- Single base image (`node:18`), which means no second base image like `nginx:alpine`.
- After building the React app (`npm run build`), it uses the default Nginx installed in the same image to serve the content.
- The build output is copied from the React app's build folder to Nginx's serving directory `/var/www/html/`.

#### Pros:
- Simpler as it uses only one base image.
- Fewer layers in the Docker image since you’re not switching between multiple images.

#### Cons:
- Larger final image because the build tools (Node.js) and runtime tools (Nginx) are combined into a single image.
- Less separation between build and runtime, making it harder to debug or optimize the image.

### 2. Multi-Stage Build Dockerfile

A multi-stage build Dockerfile allows you to separate the build and runtime environments. This results in a smaller final image, as only the necessary files are copied into the final image.

#### Dockerfile:
```Dockerfile
# Stage 1: Build React app
FROM node:18-alpine AS build

# Set working directory
WORKDIR /app

# Copy package.json
COPY package.json .

# Install dependencies
RUN npm install

# Copy all files
COPY . .

# Build the React app for production
RUN npm run build

# Stage 2: Serve the app using Nginx
FROM nginx:alpine

# Copy the built app from the build stage to Nginx's public directory
COPY --from=build /app/build /usr/share/nginx/html

# Expose port 80 for the app
EXPOSE 80

# Start Nginx to serve the app
CMD ["nginx", "-g", "daemon off;"]
```

#### Pros:
- More efficient, as it separates the build and runtime environments.
- Smaller image size since unnecessary files (like development dependencies) aren’t included in the final image.
- Easier to manage and optimize layers.

#### Cons:
- Slightly more complex than a single build.
- Multi-stage builds can take longer to set up and understand for beginners.

### 3. Development and Production Multi-Stage Dockerfile

If you want to handle both development and production environments, you can create a Dockerfile that can be switched between development and production based on a build argument.

#### Dockerfile:
```Dockerfile
# Build stage
FROM node:18-alpine AS build

WORKDIR /app

COPY package.json ./

RUN npm install

COPY . .

# Build the React app for production or development
ARG ENV=production
RUN if [ "$ENV" = "production" ]; then npm run build; else npm start; fi

# Production stage
FROM nginx:alpine AS production

COPY --from=build /app/build /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

# Development stage
FROM node:18-alpine AS development

WORKDIR /app

COPY --from=build /app /app

EXPOSE 3000
CMD ["npm", "start"]
```

#### Pros:
- Supports both production and development modes with a single Dockerfile.
- Flexible for various environments.

#### Cons:
- Increased complexity in the Dockerfile.
- Larger image for development mode since it includes the development tools and dependencies.

---

## Using Nginx as a Reverse Proxy

Yes, you can use Nginx to serve your React application. To do this, you will need to build your React application and then use Nginx to serve the static files. Here is how you can create a multi-stage Dockerfile to achieve this:

### Multi-Stage Dockerfile for React Application with Nginx

```dockerfile
# Stage 1: Build the React application
FROM node:14-alpine AS build

# Set environment variable for the application home directory
ENV APP_HOME=/usr/src/app

# Set the working directory inside the container
WORKDIR $APP_HOME

# Copy the package.json and package-lock.json files to the working directory
COPY package.json package-lock.json ./

# Install the dependencies specified in package.json
RUN npm install

# Copy the rest of the application code to the working directory
COPY . .

# Build the React application
RUN npm run build

# Stage 2: Serve the built application using Nginx
FROM nginx:alpine

# Remove the default Nginx configuration file
RUN rm /etc/nginx/conf.d/default.conf

# Copy a custom Nginx configuration file
COPY nginx.conf /etc/nginx/conf.d

# Copy the built files from the previous stage to the Nginx working directory
COPY --from=build /usr/src/app/build /usr/share/nginx/html

# Expose port 80 to allow traffic to the Nginx server
EXPOSE 80

# Start Nginx in the foreground
CMD ["nginx", "-g", "daemon off;"]
```

### Custom Nginx Configuration File (`nginx.conf`)

Create a file named `nginx.conf` in the same directory as your Dockerfile with the following content:

```nginx
server {
    # Listen for incoming HTTP requests on port 80
    listen 80;

    # The server name (hostname) that this block responds to.
    # "localhost" means it's accessible via http://localhost in the browser.
    server_name localhost;

    # The root directory where the static files (HTML, CSS, JS) are stored.
    # This is where the web server will look for files when it receives a request.
    # In this case, it's the current directory.
    # Set the root directory to serve the built React app
    root /usr/share/nginx/html;

    # The default file to serve when a directory is accessed.
    index index.html;

    # Handle all requests
    # Serve the React app (SPA Mode)
    location / {
        # If the requested file exists, serve it; otherwise, serve index.html.
        # This is useful for SPAs (Single Page Applications) like React, where
        # the app handles routing via JavaScript rather than the backend.
        try_files $uri /index.html;
    }

    # Serve static files correctly from the 'static' directory.
    # Ensures that requests to "/static/*" files are served properly.
    location /static/ {
        root /usr/share/nginx/html;
    }

    # Improve caching for performance (for images, CSS, JavaScript, fonts, etc.).
    location ~* \.(?:ico|css|js|gif|jpe?g|png|woff2?|eot|ttf|svg)$ {
        # Set browser caching for 6 months to reduce unnecessary requests.
        expires 6M;
        
        # Disable access logging for static files to reduce log clutter.
        access_log off;

        # Add Cache-Control headers to instruct browsers to cache assets.
        # "immutable" tells the browser that the file will never change,
        # preventing unnecessary re-fetching.
        add_header Cache-Control "public, max-age=15552000, immutable";

    # Handle 404 errors gracefully
    error_page 404 /index.html;    
    }
}
```

### Explanation of Directives:

1. **Stage 1: Build the React application**
   - `FROM node:14-alpine AS build`: Use the official Node.js image based on Alpine Linux for a lightweight build environment. The `AS build` syntax names this stage "build".
   - `ENV APP_HOME=/usr/src/app`: Set an environment variable for the application home directory.
   - `WORKDIR $APP_HOME`: Set the working directory inside the container to `$APP_HOME`.
   - `COPY package.json ./`: Copy the package.json file to the working directory to install dependencies.
   - `RUN npm install`: Install the dependencies specified in package.json.
   - `COPY . .`: Copy the rest of the application code to the working directory.
   - `RUN npm run build`: Build the React application, which outputs the built files to the `build` directory.

2. **Stage 2: Serve the built application using Nginx**
   - `FROM nginx:alpine`: Use the official Nginx image based on Alpine Linux for a lightweight web server.
   - `RUN rm /etc/nginx/conf.d/default.conf`: Remove the default Nginx configuration file.
   - `COPY nginx.conf /etc/nginx/conf.d`: Copy a custom Nginx configuration file to the Nginx configuration directory.
   - `COPY --from=build /usr/src/app/build /usr/share/nginx/html`: Copy the built files from the `build` directory in the build stage to the Nginx working directory.
   - `EXPOSE 80`: Expose port 80 to allow traffic to the Nginx server.
   - `CMD ["nginx", "-g", "daemon off;"]`: Start Nginx in the foreground to keep the container running.

### Building and Running the Docker Container

1. **Build the Docker image**:
   ```bash
   docker build -t react-nginx-app .
   ```

2. **Run the Docker container**:
   ```bash
   docker run -p 80:80 react-nginx-app
   ```

Your React application should now be accessible at `http://localhost`. This setup uses Nginx to serve the static files generated by the React build process.

---

Your project structure shows that it's a React application, but there is no /static/ directory yet because React hasn't been built into a production-ready format.

🚀 Next Steps: Build the React App
Before deploying with Nginx, you need to generate the production build of your React project.

1️⃣ Build the React App
Run the following command inside your project directory:

```bash
npm run build
```

or, if using Yarn:

```bash
yarn build
```

This will create a `build/` directory, which will contain the optimized production version of your app.

2️⃣ Check the /static/ Directory
Once the build is complete, check if the `/static/` directory exists:

```bash
ls -l build/static
```

If successful, you should see subdirectories like `js/`, `css/`, and `media/` inside `/static/`.

3️⃣ Update Your Nginx Configuration (If Needed)
Now that the `/static/` directory exists, make sure your Nginx configuration points to the correct build output.

```bash
ibtisam@mint-dell:~/SilverOps/DevOps/DevOps-Tools/docker/06-ReactJSPortfolio$ npm run build

> portfolio-website@0.1.0 build
> react-scripts build

Creating an optimized production build...
  
Compiled successfully.

File sizes after gzip:

  100.59 kB  build/static/js/main.d3ddb12e.js
  7.64 kB    build/static/css/main.522fb477.css

The project was built assuming it is hosted at /.
You can control this with the homepage field in your package.json.

The build folder is ready to be deployed.
You may serve it with a static server:

  npm install -g serve
  serve -s build

Find out more about deployment here:

  https://cra.link/deployment

ibtisam@mint-dell:~/SilverOps/DevOps/DevOps-Tools/docker/06-ReactJSPortfolio$ ls -l build/
total 12
-rw-rw-r-- 1 ibtisam ibtisam 1125 Feb 10 15:26 asset-manifest.json
-rw-rw-r-- 1 ibtisam ibtisam  377 Feb 10 15:26 index.html
drwxrwxr-x 5 ibtisam ibtisam 4096 Feb 10 15:26 static
ibtisam@mint-dell:~/SilverOps/DevOps/DevOps-Tools/docker/06-ReactJSPortfolio$ tree build/
build/
├── asset-manifest.json
├── index.html
└── static
    ├── css
    │   ├── main.522fb477.css
    │   └── main.522fb477.css.map
    ├── js
    │   ├── main.d3ddb12e.js
    │   ├── main.d3ddb12e.js.LICENSE.txt
    │   └── main.d3ddb12e.js.map
    └── media
        ├── cv.1676dcdc83730930d888.pdf
        ├── me.821ecddcb9953b46bf75.png
        ├── me-about.fb4452e7f286d3636ef9.jpg
        ├── portfolio1.f5e72352e5aa840702b8.jpg
        ├── portfolio2.ae187bc4e43ae6dbc263.jpg
        ├── portfolio3.cbab42559fe73e73906f.jpg
        ├── portfolio4.6bf60cbb8fe5f7d7af5d.jpg
        ├── portfolio5.a82ae9a163c1ac5e3a07.png
        └── portfolio6.72d1107e313049cf15a7.jpg

5 directories, 16 files
ibtisam@mint-dell:~/SilverOps/DevOps/DevOps-Tools/docker/06-ReactJSPortfolio$ 
```

Your React build was successful 🎉, and now you do have a `/static/` directory inside `build/`. This means your static assets (CSS, JS, images, etc.) are ready to be served.

✅ Next Steps: Deploy with Nginx
Now, you need to update your Nginx configuration to serve the React app properly.

Modify `nginx.config`
Open your `nginx.config` and update it with the following:

```nginx
server {
    listen 80;
    server_name localhost;

    # Set the root directory to serve the built React app
    root /usr/share/nginx/html;
    index index.html;

    # Serve the React app (SPA Mode)
    location / {
        try_files $uri /index.html;
    }

    # Serve static files efficiently
    location /static/ {
        root /usr/share/nginx/html;
        expires 1y;
        access_log off;
    }

    # Handle 404 errors gracefully
    error_page 404 /index.html;
}
```
---

Your error message:

```bash
cp: can't create '/usr/share/nginx/html/asset-manifest.json': No such file or directory
```

indicates that the `/usr/share/nginx/html/` directory doesn't exist inside your container at the time of copying the build files.

✅ **Solution: Ensure Nginx Directory Exists Before Copying**
Modify your Dockerfile to explicitly create the directory before copying files.

**Fix your Dockerfile (Modify this section)**
Replace:

```dockerfile
RUN cp -r /app/build/* /usr/share/nginx/html/
```

with:

```dockerfile
RUN mkdir -p /usr/share/nginx/html && cp -r /app/build/* /usr/share/nginx/html/
```

✅ **Alternative Fix: Use `/var/www/html/` Instead of `/usr/share/nginx/html/`**
Some Alpine-based Nginx images expect web files to be inside `/var/www/html/`. If the previous fix doesn’t work, update:

```dockerfile
RUN mkdir -p /var/www/html && cp -r /app/build/* /var/www/html/
```

and modify your Nginx config to serve from `/var/www/html/`:

```nginx
server {
    listen 80;
    server_name localhost;

    root /var/www/html;
    index index.html;
    
    location / {
        try_files $uri /index.html;
    }
}
```

### **Rebuild and Restart the Container**
After making changes, rebuild and restart your container:

```bash
docker compose down
docker compose up -d --build
```
---

The error:

```nginx
nginx: [emerg] "server" directive is not allowed here in /etc/nginx/conf.d/default.conf:1
```

suggests that your `nginx.conf` file has an incorrect syntax or is being placed in the wrong location.

✅ **Solution: Ensure Correct Nginx Configuration Format**
Open your `nginx.conf` file inside your project and ensure it follows the correct format.

### 1️⃣ Check `nginx.conf` Placement
Make sure you're copying `nginx.conf` to the correct directory:

**Correct:**
```dockerfile
COPY nginx.conf /etc/nginx/nginx.conf
```

**Incorrect:**
```dockerfile
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

⚡ **Why?**
- `/etc/nginx/nginx.conf` is the main configuration file.
- `/etc/nginx/conf.d/default.conf` is a server block file, so it should only contain the `server {}` block.

### 2️⃣ If Keeping `default.conf`, Ensure It Has a `server` Block
If you must copy it to `/etc/nginx/conf.d/default.conf`, the file should look like this:

```nginx
server {
    listen 80;
    server_name localhost;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri /index.html;
    }
}
```

⚠ **Remove any top-level `http {}` block from `default.conf`**, as it's only needed in `nginx.conf`.

### 🔍 Possible Issues & Fixes
#### 1️⃣ `nginx.conf` is Misplaced
Check where you're copying `nginx.conf` inside your Dockerfile.

✅ **Fix: Use the Correct Destination**
Instead of:
```dockerfile
COPY nginx.conf /etc/nginx/conf.d/default.conf
```
Use:
```dockerfile
COPY nginx.conf /etc/nginx/nginx.conf
```

⚠️ **Why?**
- `nginx.conf` should be placed at `/etc/nginx/nginx.conf` because it's the main configuration file.
- `default.conf` (inside `/etc/nginx/conf.d/`) should contain only `server {}` blocks.

#### 2️⃣ Remove Any Existing `default.conf`
If an incorrect default file exists, it might override your settings.
Run the following in your Dockerfile to remove it:

```dockerfile
RUN rm -f /etc/nginx/conf.d/default.conf
```

Then rebuild and restart:

```bash
docker compose down
docker compose up -d --build
```

#### 3️⃣ Verify Your Container’s `nginx.conf`
After the container is running, check if Nginx is using the right config:

```bash
docker exec -it 06-reactjsportfolio-portfolio-1 cat /etc/nginx/nginx.conf
```

If the output does not match your intended config, it's not being copied correctly.

### ✅ Final Steps
1. Ensure your Dockerfile copies `nginx.conf` correctly.
2. Remove any existing `/etc/nginx/conf.d/default.conf`.
3. Rebuild and restart the container.
4. Check logs:

```bash
docker logs -f 06-reactjsportfolio-portfolio-1
```

Your container should now start successfully! 🚀 Let me know if you need further help.

This error occurs when the `nginx.conf` file is being used as the main Nginx configuration file but contains a `server {}` block, which should be inside the `conf.d/` or `http.d/` directory as a separate site configuration.

## **🔹 Solution: Ensure Proper Nginx Configuration Placement**

### **1️⃣ Update Dockerfile**
Modify this line in your Dockerfile:

**Incorrect:**
```dockerfile
COPY nginx.conf /etc/nginx/nginx.conf
```

**Correct:**
```dockerfile
COPY nginx.conf /etc/nginx/conf.d/default.conf
```
This ensures that Nginx treats your `nginx.conf` as a site configuration instead of the main config file.

### **2️⃣ Rebuild and Restart the Container**
Run the following commands:
```sh
docker compose down
docker compose up -d --build
```
This should fix the error.

## **✅ How These Files Should Look**
### **Main Configuration: `/etc/nginx/nginx.conf`**
```nginx
events {}  # Required for Nginx

http {
    include /etc/nginx/conf.d/*.conf;  # Include all configs in conf.d/
}
```
### **Custom Configuration: `/etc/nginx/conf.d/default.conf`**
```nginx
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri /index.html;
    }

    location /static/ {
        root /usr/share/nginx/html;
    }

    location ~* \.(?:ico|css|js|gif|jpe?g|png|woff2?|eot|ttf|svg)$ {
        expires 6M;
        access_log off;
        add_header Cache-Control "public, max-age=15552000, immutable";
    }

    error_page 404 /index.html;
}
```

### **3️⃣ Verify Configuration Inside the Container**
After restarting, check if Nginx is using the correct configuration:
```sh
docker exec -it <container_id> cat /etc/nginx/nginx.conf
```
If the output does not match the intended config, it's not being copied correctly.

---

## **📌 Special Case: Alpine-based Nginx Images**
For Alpine-based Nginx images, configurations must be placed in `/etc/nginx/http.d/` instead of `/etc/nginx/conf.d/`.

### **🔹 Fix for Alpine: Move Server Config to `/etc/nginx/http.d/`**

### **1️⃣ Main Configuration: `/etc/nginx/nginx.conf`**
```nginx
events {}

http {
    include /etc/nginx/http.d/*.conf;  # Alpine uses `http.d/` instead of `conf.d/`
}
```

### **2️⃣ Server Configuration: `/etc/nginx/http.d/default.conf`**
```nginx
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri /index.html;
    }

    location /static/ {
        root /usr/share/nginx/html;
    }

    location ~* \.(?:ico|css|js|gif|jpe?g|png|woff2?|eot|ttf|svg)$ {
        expires 6M;
        access_log off;
        add_header Cache-Control "public, max-age=15552000, immutable";
    }

    error_page 404 /index.html;
}
```

### **3️⃣ Update Dockerfile for Alpine**
```dockerfile
COPY nginx.conf /etc/nginx/http.d/default.conf  # Alpine expects it in `http.d/`
```

## **Final Steps**

### **1️⃣ Rebuild and Restart the Container**
```sh
docker compose down
docker compose up -d --build
```

### **2️⃣ Verify Configuration Inside the Container**
```sh
docker exec -it <container_id> ls /etc/nginx/http.d/
```
Expected output:
```sh
default.conf
```

### **3️⃣ Check Logs for Any Errors**
```sh
docker logs -f <container_id>
```

**This should resolve your Nginx configuration issues! 🚀**


