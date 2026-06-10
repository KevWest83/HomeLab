services:
  navidrome:
    image: deluan/navidrome:latest
    user: 1000:1000
    ports:
      - "4533:4533"
    restart: unless-stopped
    volumes:
      - "./data:/data"
      - "/media/entertainment_pool/Music:/music:ro"
