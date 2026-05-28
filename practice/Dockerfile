# Your Linux lab — a Debian "server" you grow over the course.
# Add packages here as lessons ask you to, then rebuild:
#   docker compose up -d --build
FROM debian:12-slim

# A base toolkit so the box is usable from lesson 01.
# (Later lessons will have you `apt install` more — that's intentional practice.)
RUN apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        wget \
        less \
        nano \
        vim \
        procps \
        iproute2 \
        iputils-ping \
        net-tools \
        tree \
        htop \
        man-db \
        sudo \
        bash-completion \
    && rm -rf /var/lib/apt/lists/*

# Persistent working area — bind-mounted from ./server-data on your Mac.
# Anything you create here survives `docker compose down`/`up`.
RUN mkdir -p /work
WORKDIR /work

# Keep the container alive so you can `docker compose exec server bash` into it.
# (A container lives only as long as PID 1 — see lesson 21.)
CMD ["sleep", "infinity"]
