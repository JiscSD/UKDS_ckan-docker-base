ARG ENV=base

FROM python:3.12-slim-bookworm AS python


# ┌─────────────────────────────────────────────────────────────┐
# │                                                             │
# │ Base image (Production env)                                 │
# │ ---------------------------                                 │
# │ This is always built                                        │
# │                                                             │
# └─────────────────────────────────────────────────────────────┘


FROM python AS base

# Tag passed through via the Makefile
ARG CKAN_REF=${CKAN_REF}
# Make it available to child images
ENV CKAN_REF=${CKAN_REF}
ENV CKAN_INSTALL_VERSION=2.11

# Internals, you probably don't need to change these
ENV TZ=EN_GB
ENV APP_DIR=/srv/app
ENV SRC_DIR=${APP_DIR}/src
ENV CKAN_INI=${APP_DIR}/ckan.ini
ENV PIP_SRC=${SRC_DIR}
ENV CKAN_STORAGE_PATH=/var/lib/ckan
ENV GIT_URL=https://github.com/ckan/ckan.git

# Customize these in the environment (.env) file if needed
ENV CKAN_SITE_URL=http://localhost:5000
ENV CKAN__PLUGINS="image_view text_view datatables_view datastore envvars"

# UWSGI options
ENV UWSGI_HARAKIRI=50

WORKDIR ${APP_DIR}

# Check for updates
RUN apt-get update && \
    apt-get upgrade -y

# Set up timezone
RUN echo ${TZ} > /etc/timezone

# Set LC_ALL=en_GB.UTF-8 will ensure that all locale-dependent operations in the current environment
# will use English language and United States cultural conventions with UTF-8 character encoding
ENV LC_ALL=en_GB.UTF-8

# Set the locale
RUN apt-get update && \
    apt-get install --no-install-recommends -y locales && \
    apt-get clean && \
    sed -i "/$LC_ALL/s/^# //g" /etc/locale.gen && \
    dpkg-reconfigure --frontend=noninteractive locales && \
    update-locale LANG=${LC_ALL}

# Install system libraries
RUN apt-get install --no-install-recommends -y \
        apt-utils \
        g++ \
        git \
        libpq-dev \
        libtool \
        linux-headers-generic \
        mc \
        nano \
        wget && \
    apt-get clean

# Create the src directory
RUN mkdir -p ${SRC_DIR}

# Install uwsgi, the CKAN application, the dependency packages for CKAN plus some confiquration
RUN python3 -m pip install -U pip && \
    python3 -m pip install uwsgi && \
    python3 -m pip install wheel && \
    cd ${SRC_DIR} && \
    python3 -m pip install -e git+${GIT_URL}@${CKAN_INSTALL_VERSION}#egg=ckan && \
    python3 -m pip install --no-binary markdown -r ${SRC_DIR}/ckan/requirements.txt && \
    # Install CKAN Plugins including envvars to support loading config from environment variables
    python3 -m pip install -e git+https://github.com/ckan/ckanext-pages.git#egg=ckanext-pages && \
    python3 -m pip install ckanext-envvars \
            ckanext-geoview && \
    python3 -m pip install ckanext-xloader && \
    python3 -m pip install -r https://raw.githubusercontent.com/ckan/ckanext-xloader/master/requirements.txt && \
    python3 -m pip install -e git+https://github.com/JiscSD/ckanext-matomo-js.git#egg=ckanext-matomo-js && \
    python3 -m pip install -e git+https://github.com/ckan/ckanext-harvest.git#egg=ckanext-harvest && \
    python3 -m pip install -r ${SRC_DIR}/ckanext-harvest/requirements.txt && \
    python3 -m pip install -e git+https://github.com/ckan/ckanext-dcat.git#egg=ckanext-dcat && \
    python3 -m pip install -r ${SRC_DIR}/ckanext-dcat/requirements.txt && \
    python3 -m pip install -e git+https://github.com/JiscSD/ckanext-customised_fields_from_tag_vocabulary.git#egg=ckanext-customised_fields_from_tag_vocabulary && \
    python3 -m pip install -e git+https://github.com/JiscSD/ckanext-ukdstheme.git#egg=ckanext-ukdstheme && \
    python3 -m pip install -r ${SRC_DIR}/ckanext-ukdstheme/requirements.txt && \
    python3 -m pip install -e git+https://github.com/JiscSD/ckanext-dataset_geo_display.git#egg=ckanext-dataset_geo_display && \
    python3 -m pip install -r ${SRC_DIR}/ckanext-dataset-geo-display/requirements.txt && \
    python3 -m pip install -e git+https://github.com/JiscSD/ckanext-ukds_custom_dcat_profiles.git#egg=ckanext-ukds_custom_dcat_profiles && \
    python3 -m pip install -r ${SRC_DIR}/ckanext-ukds-custom-dcat-profiles/requirements.txt && \
    python3 -m pip install -e git+https://github.com/JiscSD/ckanext-cookie-banner.git#egg=ckanext-cookie-banner && \
    # Create and update CKAN config
    ckan generate config ${CKAN_INI} && \
    ckan config-tool ${CKAN_INI} "SECRET_KEY = " && \
    ckan config-tool ${CKAN_INI} "ckan.plugins = ${CKAN__PLUGINS}"

# Create ckan and ckan-sys users and the ckan-sys group plus set up the storage path
RUN groupadd -g 502 ckan-sys && \
    useradd -rm -d /srv/app -s /bin/bash -g ckan-sys -u 502 ckan-sys && \
    useradd -rm -d /srv/app -s /bin/bash -g ckan-sys -u 503 ckan

COPY setup/prerun.py ${APP_DIR}
COPY setup/start_ckan.sh ${APP_DIR}
ADD https://raw.githubusercontent.com/ckan/ckan/${CKAN_INSTALL_VERSION}/wsgi.py ${APP_DIR}

RUN chmod 644 ${APP_DIR}/wsgi.py
# Create entrypoint directory for children image scripts
RUN mkdir -p /docker-entrypoint.d && \
    chmod 755 /docker-entrypoint.d


# Set the ownership of the app directory, usr/local and the entrypoint directory to the ckan-sys user
RUN chown -R ckan-sys:ckan-sys ${APP_DIR} && \
    chown -R ckan-sys:ckan-sys /docker-entrypoint.d && \
    chown -R ckan-sys:ckan-sys /usr/local

# Set the ownership of the CKAN config file, src and the storage path to the ckan user
RUN chown ckan:ckan-sys ${APP_DIR}/ckan.ini && \
    chown -R ckan:ckan-sys ${APP_DIR}/src && \
    mkdir -p ${CKAN_STORAGE_PATH} && \
    chown -R ckan:ckan-sys ${CKAN_STORAGE_PATH}

USER ckan

EXPOSE 5000

CMD ["/srv/app/start_ckan.sh"]


# ┌─────────────────────────────────────────────────────────────┐
# │                                                             │
# │ Dev image (Development env)                                 │
# │ ---------------------------                                 │
# │ This is only built when ENV=dev                             │
# │                                                             │
# └─────────────────────────────────────────────────────────────┘


FROM base AS dev

ENV SRC_EXTENSIONS_DIR=${APP_DIR}/src_extensions

USER root

WORKDIR ${SRC_DIR}/ckan
RUN python3 -m pip install -r https://raw.githubusercontent.com/ckan/ckan/${CKAN_INSTALL_VERSION}/dev-requirements.txt

COPY --chown=ckan-sys:ckan-sys setup/unsafe.cert setup/unsafe.key setup/start_ckan_development.sh setup/install_src.sh ${APP_DIR}

# Update local directories
RUN mkdir -p ${SRC_EXTENSIONS_DIR} /var/lib/ckan && \
    chown -R ckan-sys:ckan-sys ${SRC_EXTENSIONS_DIR} && \
    chown -R ckan:ckan-sys /var/lib/ckan/ && \
    chmod 775 ${SRC_EXTENSIONS_DIR}

USER ckan

CMD ["/srv/app/start_ckan_development.sh"]


# ──────────────────────────────────────────────────────────────


FROM ${ENV} AS final
