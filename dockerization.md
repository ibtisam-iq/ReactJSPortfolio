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
```