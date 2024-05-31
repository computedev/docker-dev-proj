FROM alpine as cmatbuilder
WORKDIR /cmatrix/
RUN apk update --no-cache && \
apk add autoconf automake alpine-sdk ncurses-dev ncurses-static && \
apk add git && \ 
git clone https://github.com/spurin/cmatrix.git . && \
autoreconf -i && \
mkdir -p /usr/lib/kbd/consolefonts /usr/share/consolefonts && \
./configure LDFLAGS="-static" && \
make

from alpine
RUN apk update --no-cache && apk add ncurses-terminfo-base && \
adduser -g "niranjan " -s /user/sbin/nologin -D -H nir
USER nir
COPY --from=cmatbuilder /cmatrix/cmatrix /cmatrix
ENTRYPOINT ["./cmatrix"]
# This will overide the entrypoint if any commandline argument is passed
CMD ["-b"]