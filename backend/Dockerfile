FROM arm64v8/alpine:3.6 as BUILDER

ENV TAIGA_VERSION 3.1.3

ensure local python is preferred over distribution python
ENV PATH /usr/local/bin:$PATH

# http://bugs.python.org/issue19846
# > At the moment, setting "LANG=C" on a Linux system *fundamentally breaks Python 3*, and that's not OK.
ENV LANG C.UTF-8

# install ca-certificates so that HTTPS works consistently
# the other runtime dependencies for Python are installed later
RUN apk add --no-cache ca-certificates

ENV GPG_KEY 0D96DF4D4110E5C43FBFB17F2D347EA6AA65421D
ENV PYTHON_VERSION 3.6.5

RUN set -ex \
 && apk add --no-cache --virtual .fetch-deps \
   gnupg \
   libressl \
   tar \
   xz \
 \
 && wget -O python.tar.xz "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz" \
 && wget -O python.tar.xz.asc "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz.asc" \
 && export GNUPGHOME="$(mktemp -d)" \
 && gpg --keyserver ha.pool.sks-keyservers.net --recv-keys "$GPG_KEY" \
 && gpg --batch --verify python.tar.xz.asc python.tar.xz \
 && rm -rf "$GNUPGHOME" python.tar.xz.asc \
 && mkdir -p /usr/src/python \
 && tar -xJC /usr/src/python --strip-components=1 -f python.tar.xz \
 && rm python.tar.xz \
 \
 && apk add --no-cache --virtual .build-deps  \
   bzip2-dev \
   coreutils \
   dpkg-dev dpkg \
   expat-dev \
   gcc \
   gdbm-dev \
   libc-dev \
   libffi-dev \
   linux-headers \
   make \
   ncurses-dev \
   libressl \
   libressl-dev \
   pax-utils \
   readline-dev \
   sqlite-dev \
   tcl-dev \
   tk \
   tk-dev \
   xz-dev \
   zlib-dev \
# add build deps before removing fetch deps in case there's overlap
 && apk del .fetch-deps \
 \
 && cd /usr/src/python \
 && gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)" \
 && ./configure \
   --build="$gnuArch" \
   --enable-loadable-sqlite-extensions \
   --enable-shared \
   --with-system-expat \
   --with-system-ffi \
   --without-ensurepip \
 && make -j "$(nproc)" \
# set thread stack size to 1MB so we don't segfault before we hit sys.getrecursionlimit()
# https://github.com/alpinelinux/aports/commit/2026e1259422d4e0cf92391ca2d3844356c649d0
   EXTRA_CFLAGS="-DTHREAD_STACK_SIZE=0x100000" \
 && make install \
 \
 && runDeps="$( \
   scanelf --needed --nobanner --format '%n#p' --recursive /usr/local \
     | tr ',' '\n' \
     | sort -u \
     | awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
 )" \
 && apk add --virtual .python-rundeps $runDeps \
 && apk del .build-deps \
 \
 && find /usr/local -depth \
   \( \
     \( -type d -a \( -name test -o -name tests \) \) \
     -o \
     \( -type f -a \( -name '*.pyc' -o -name '*.pyo' \) \) \
   \) -exec rm -rf '{}' + \
 && rm -rf /usr/src/python

# make some useful symlinks that are expected to exist
RUN cd /usr/local/bin \
 && ln -s idle3 idle \
 && ln -s pydoc3 pydoc \
 && ln -s python3 python \
 && ln -s python3-config python-config

# if this is called "PIP_VERSION", pip explodes with "ValueError: invalid truth value '<VERSION>'"
ENV PYTHON_PIP_VERSION 9.0.3

RUN set -ex; \
 \
 apk add --no-cache --virtual .fetch-deps libressl; \
 \
 wget -O get-pip.py 'https://bootstrap.pypa.io/get-pip.py'; \
 \
 apk del .fetch-deps; \
 \
 python get-pip.py \
   --disable-pip-version-check \
   --no-cache-dir \
   "pip==$PYTHON_PIP_VERSION" \
 ; \
 pip --version; \
 \
 find /usr/local -depth \
   \( \
     \( -type d -a \( -name test -o -name tests \) \) \
     -o \
     \( -type f -a \( -name '*.pyc' -o -name '*.pyo' \) \) \
   \) -exec rm -rf '{}' +; \
rm -f get-pip.py

RUN apk update \
    && apk add --virtual build-dependencies \
    # Essentials
    gcc musl-dev libressl-dev \
    # Postgres python client (psycopg2) build dependency
    postgresql-dev \
    # Pillow / PIL build dependencies
    freetype-dev jpeg-dev libwebp-dev tiff-dev libpng-dev lcms2-dev \
    openjpeg-dev zlib-dev libxslt-dev libxml2-dev libffi-dev

WORKDIR /taiga_backend

ADD https://github.com/taigaio/taiga-back/archive/${TAIGA_VERSION}.tar.gz ./
RUN tar -xzf ${TAIGA_VERSION}.tar.gz -C ./ taiga-back-${TAIGA_VERSION} --strip-components=1
RUN rm ${TAIGA_VERSION}.tar.gz

# Don't want pip to use git, so I'm replacing with pypi.
RUN sed -i 's,git+https://github.com/Xof/django-pglocks.git,django-pglocks==1.0.2,g' requirements.txt

# local.py and checkdb.py and celery
# using gevent to run taiga gunicorn (workers)
# using gevent on celery (workers)
RUN echo "django-environ==0.4.0" >> requirements.txt
RUN echo "gevent==1.1.2" >> requirements.txt
RUN echo "django-anymail==0.5" >> requirements.txt

RUN pip wheel --wheel-dir=./taiga_python_dependencies -r requirements.txt


# FINAL IMAGE
FROM python:3.6-alpine3.6

LABEL maintainer="douglasmirandasilva@gmail.com"
LABEL taiga_version="tag:3.1.3"

WORKDIR /taiga_backend

COPY --from=BUILDER /taiga_backend ./

RUN apk add --no-cache \
    # Pillow / PIL
    freetype jpeg libwebp tiff libpng lcms2 openjpeg zlib libxslt libxml2 libffi \
    # Postgres python client
    libpq \
    # Needed for localization stuff: python manage.py compilemessages
    gettext

RUN pip install --no-cache-dir --no-index --find-links=taiga_python_dependencies -r requirements.txt \
    && rm -R ./taiga_python_dependencies \
    && python manage.py compilemessages

COPY local.py settings/local.py
COPY celery_local.py settings/celery_local.py
COPY scripts/ /scripts/

RUN mkdir /taiga_backend/media && mkdir /taiga_backend/static-root \
    && addgroup -S taiga && adduser -S -G taiga taiga \
    && chown -R taiga ./ \
    && chown -R taiga /scripts/ \
    && chmod +x /scripts/entrypoint.sh

USER taiga
