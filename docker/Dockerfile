# Custom Traccar image built from YOUR forks.
#
# Build context note: this Dockerfile expects to be built from a recursive
# checkout of your `traccar` fork (which contains `traccar-web` as a submodule).
# In CI we check the fork out with submodules and build from there; this file is
# kept in traccar-infra for reference / local builds via `--build-context`.
#
# Stage 1: build the web bundle (React + Vite)
FROM node:22-alpine AS web
WORKDIR /web
# traccar-web is the submodule directory inside the traccar fork checkout.
# .npmrc carries the project's npm policy (legacy-peer-deps) and must be present
# for `npm ci` to resolve a tree matching the lockfile.
COPY traccar-web/package*.json traccar-web/.npmrc ./
RUN npm ci
COPY traccar-web/ ./
# Branding assets (logo, favicon, etc.) are overlaid before build so they are
# baked into the bundle. Keep cosmetic branding here; see /branding in infra repo.
COPY branding/web/ ./public/
RUN npm run build

# Stage 2: build the server JAR (Gradle)
FROM eclipse-temurin:21-jdk AS server
WORKDIR /src
COPY . .
# Drop the freshly built web bundle into the location the server serves from
COPY --from=web /web/build ./traccar-web/build
RUN ./gradlew --no-daemon assemble

# Stage 3: runtime — start from the official image to inherit its JRE + layout,
# then swap in our compiled artifacts. This keeps us aligned with upstream's
# runtime while shipping our customised binaries.
FROM traccar/traccar:6-alpine AS runtime
# Replace server jar
COPY --from=server /src/target/tracker-server.jar /opt/traccar/tracker-server.jar
# Replace web assets
COPY --from=web /web/build /opt/traccar/web
# Default config can be overridden by a host-mounted traccar.xml at runtime
EXPOSE 8082
# Device protocol ports are published at `docker run`/compose time, not here.
