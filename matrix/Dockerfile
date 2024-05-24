FROM alpine
WORKDIR /cmatrix/
RUN echo ls -l 
RUN apk update 
RUN apk add git 
RUN git clone https://github.com/spurin/cmatrix.git .

RUN apk add pkgconf
RUN apk add autoconf

RUN apk add autoconf

RUN apk add automake
RUN echo $?
RUN autoreconf -i
RUN apk add alpine-sdk
RUN apk add ncurses-dev ncurses-static
RUN mkdir -p /usr/lib/kbd/consolefonts /usr/share/consolefonts

RUN ./configure LDFLAGS="-static"

RUN make
CMD ["./cmatrix"]