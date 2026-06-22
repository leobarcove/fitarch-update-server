# FitArch self-hosted Expo update server (Next.js).
# The private key and the updates payload are provided at runtime as volumes,
# never baked into the image — so the image carries no secrets and no bundles.
FROM node:20-alpine

WORKDIR /app

# Install dependencies first for better layer caching.
COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile

# Build the Next.js app.
COPY . .
RUN yarn build

# Next listens on 3000; Traefik terminates TLS and forwards plain HTTP here.
EXPOSE 3000

# `yarn start` runs `next start` with PRIVATE_KEY_PATH already set in package.json.
# PUBLIC_BASE_URL is injected by compose so manifest asset URLs are absolute + HTTPS.
CMD ["yarn", "start"]
