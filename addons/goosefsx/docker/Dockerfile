FROM ubuntu:jammy
RUN apt update && \
    apt install --yes libnfs13 libfuse2 fuse python3 bash && \
    apt clean autoclean && \
    apt autoremove --yes && \
    rm -rf /var/lib/{apt,dpkg,cache,log}/
ADD ./fluid_config_init.py /
ADD ./entrypoint.sh /usr/local/bin
CMD ["/usr/local/bin/entrypoint.sh"]