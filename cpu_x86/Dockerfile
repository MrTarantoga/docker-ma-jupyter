ARG ROOT_CONTAINER=ubuntu:22.04

FROM $ROOT_CONTAINER

LABEL maintainer="My Data Science Jupyter notebook <goetz-dev@web.de>"
ARG NB_USER="jupyter"
ARG NB_UID="1000"
ARG NB_GID="100"

# Fix: https://github.com/hadolint/hadolint/wiki/DL4006
# Fix: https://github.com/koalaman/shellcheck/wiki/SC3014
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

USER root

# Install all OS dependencies for notebook server that starts but lacks all
# features (e.g., download as all possible file formats)
ENV DEBIAN_FRONTEND noninteractive
ENV PYTHON_DEPENDENCIES_RUN="openssl lzma libncurses5 tk uuid libeccodes0"
ENV PYTHON_DEPENDENCIES_BUILD="build-essential lcov pkg-config \
    libbz2-dev libffi-dev libgdbm-dev libgdbm-compat-dev liblzma-dev \
    libncurses5-dev libreadline6-dev libsqlite3-dev libssl-dev \
    lzma-dev tk-dev uuid-dev zlib1g-dev git"

ENV PYTHON_BUILD_COMMAND="git clone --branch v3.10.8 --single-branch --depth 1 https://github.com/python/cpython.git && \
    cd cpython && \
    ./configure --enable-optimizations && \
    make -j && \
    make install && \
    cd .. && \
    rm -rf cpython && \
    python3 -m ensurepip --upgrade && \
    pip3 install --upgrade pip"

ENV NODEJS_BUILD_COMMAND="git clone --branch v19.0.0 --single-branch --depth 1 https://github.com/nodejs/node.git && \
    cd node && \
    ./configure && \
    make -j4 && \
    make install && \
    cd .. && \
    rm -rf node"

ENV TENSORFLOW="\
    tensorflow-gpu==2.9.2 \
    tensorboard \
"

ENV PYTROCH="\
    torch \
    gpytroch \
"

ENV FORMER_PROJ_PACKAGES="\
    tabulate \
    plotly \
    xarray \
    eccodes \
    cfgrib \
"

ENV MA_PYPI_PACKAGES="\
    ${FORMER_PROJ_PACKAGES} \
    ${PYTROCH} \
    scipy \
    scikit-learn \
    xgboost \
    numpy \
    zstandard \
    pandas \
    dask[dataframe] \
    tables \
    geopandas \
    seaborn \
    statsmodels \
    matplotlib \
    Pillow \
    pydot \
    h5py \
    hdf5plugin \
    netCDF4 \
    jenkspy \
    bayesian-optimization \
    joblib \
"

ENV INSTALL_PYTHON_PACKAGES="pip3 install wheel \
    notebook \
    jupyterhub \
    jupyterlab \
    jupyterlab-lsp \
    vega \
    python-lsp-server[all] \
    tqdm \
    ipywidgets \
    ${MA_PYPI_PACKAGES} \
"

ENV CLEANUP_PYTHON_INSTALL="pip3 cache purge"

RUN apt-get update --yes && \
    # - apt-get upgrade is run to patch known vulnerabilities in apt-get packages as
    #   the ubuntu base image is rebuilt too seldom sometimes (less than once a month)
    apt-get upgrade --yes && \
    apt-get install --yes --no-install-recommends \
    fonts-liberation \
    ca-certificates \
    locales \
    sudo \
    htop \
    # - pandoc is used to convert notebooks to html files
    #   it's not present in arm64 ubuntu image, so we install it here
    pandoc \
    # - graphviz is used to create tensorflow picture of the graph
    graphviz \
	# - texlive-full - is used to generate with jupyter latex documents and pdf export
	texlive-full \
    # - tini is installed as a helpful container entrypoint that reaps zombie
    #   processes and such of the actual executable we want to start, see
    #   https://github.com/krallin/tini#why-tini for details.
    tini \
    # - wget is installed as dependecy for the healthcheck task
    wget \
    ${PYTHON_DEPENDENCIES_RUN} \
    ${PYTHON_DEPENDENCIES_BUILD} &&\
    apt-get clean && \
    echo "en_US.UTF-8 UTF-8" > /etc/locale.gen && \
    locale-gen && \
    eval ${PYTHON_BUILD_COMMAND} && \
    eval ${NODEJS_BUILD_COMMAND} && \
    eval ${INSTALL_PYTHON_PACKAGES} && \
    eval ${CLEANUP_PYTHON_INSTALL} && \
    # clean installation
    apt-get purge -y ${PYTHON_DEPENDENCIES_BUILD} && \
    rm -rf /usr/local/include && \
    rm -rf /usr/include

# Activate extensions of jupyter
RUN pip3 install jupyter_contrib_nbextensions && \
    jupyter contrib nbextension install --system && \
    jupyter nbextension enable codefolding/main --sys-prefix && \
    jupyter nbextension enable --py widgetsnbextension --sys-prefix

# Configure environment
ENV SHELL=/bin/bash \
    NB_USER="${NB_USER}" \
    NB_UID=${NB_UID} \
    NB_GID=${NB_GID} \
    LC_ALL=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    LANGUAGE=en_US.UTF-8
ENV HOME="/home/${NB_USER}"

# Create NB_USER with name jupyter user with UID=1000 and in the 'users' group
# and make sure these dirs are writable by the `users` group.
RUN echo "auth requisite pam_deny.so" >> /etc/pam.d/su && \
    sed -i.bak -e 's/^%admin/#%admin/' /etc/sudoers && \
    sed -i.bak -e 's/^%sudo/#%sudo/' /etc/sudoers && \
    useradd -l -m -s /bin/bash -N -u "${NB_UID}" "${NB_USER}" && \
    chmod g+w /etc/passwd && \
    chown jupyter:users -R "${HOME}"

USER ${NB_UID}

# Setup work directory for backward-compatibility
RUN mkdir "/home/${NB_USER}/work" && \
    chown jupyter:users -R "/home/${NB_USER}"

EXPOSE 8888

# Configure container startup
ENTRYPOINT ["tini", "-g", "--"]
CMD ["jupyter", "lab", "--port=8888", "--no-browser", "--ip=0.0.0.0"]

# Fix permissions on /etc/jupyter as root
USER root

# HEALTHCHECK documentation: https://docs.docker.com/engine/reference/builder/#healthcheck
# This healtcheck works well for `lab`, `notebook`, `nbclassic`, `server` and `retro` jupyter commands
# https://github.com/jupyter/docker-stacks/issues/915#issuecomment-1068528799
HEALTHCHECK  --interval=15s --timeout=3s --start-period=5s --retries=3 \
    CMD wget -O- --no-verbose --tries=1 --no-check-certificate \
    http${GEN_CERT:+s}://localhost:8888${JUPYTERHUB_SERVICE_PREFIX:-/}api || exit 1

# Switch back to jupyter to avoid accidental container runs as root
USER ${NB_UID}

WORKDIR "${HOME}"
