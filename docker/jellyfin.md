services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    user: "1000:1000"
    devices:
      - /dev/dri:/dev/dri
    group_add:
      - render
      - video
    volumes:
      - ./config:/config
      - ./cache:/cache
      - /media/entertainment_pool/Movies:/media/Movies
      - /media/entertainment_pool/TV Shows:/media/TVShows
      - /media/entertainment_pool/Anime:/media/Anime
      - /media/entertainment_pool/Wrestling:/media/Wrestling
    ports:
      - "8096:8096"
