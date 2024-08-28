FROM ubuntu:18.04 as base 
# FROM alpine:3.8 as base
## using alpine would be slimer but doesnt have apt-get so it would imply reworking quite a few things
## however could be bad for compatibility with the Jupyter version (eg copy-paste between Dockerfiles might not work anymore)

## Separating build & production could also help reduce image size

LABEL maintainer="ordonez"

# simcore-user uid=8004(${SC_USER_NAME}) gid=8004(${SC_USER_NAME}) groups=8004(${SC_USER_NAME})
ENV SC_USER_ID 8004
ENV SC_USER_NAME scu
# ubuntu:
RUN adduser --uid ${SC_USER_ID} --disabled-password --gecos "" --shell /bin/sh --home /home/${SC_USER_NAME} ${SC_USER_NAME}
# # alpine: 
# RUN adduser -D -u ${SC_USER_ID} -s /bin/sh -h /home/${SC_USER_NAME} ${SC_USER_NAME}


# ubuntu:
RUN apt-get update \
    && apt-get -y install --no-install-recommends \
    jq \
    && rm --recursive --force /var/lib/apt/lists/*

# # alpine:
# RUN apk add --no-cache \
#     su-exec \
#     jq

# -------------------------- Build stage -------------------
# Installs build/package management tools and third party dependencies
#
# + /build             WORKDIR
#

FROM base as build

ENV SC_BUILD_TARGET build

WORKDIR /build
# defines the output of the build
RUN mkdir -p /build/bin
# copy src code
COPY --chown=${SC_USER_NAME}:${SC_USER_NAME} src/comp-medimproc src/comp-medimproc
# ------------------------------------------------------------------------------------
#TODO:
# uncomment and adapt if build steps are necessary
# RUN cp -R src/comp-medimproc/* /build/bin
# ------------------------------------------------------------------------------------

# --------------------------Production stage -------------------
# Final cleanup up to reduce image size and startup setup
# Runs as ${SC_USER_NAME} (non-root user)
#
#  + /home/${SC_USER_NAME}     $HOME = WORKDIR
#    + comp-medimproc [${SC_USER_NAME}:${SC_USER_NAME}]
#    + docker [${SC_USER_NAME}:${SC_USER_NAME}]
#    + service.cli [${SC_USER_NAME}:${SC_USER_NAME}]
#
FROM base as production

ENV SC_BUILD_TARGET production
ENV SC_BOOT_MODE production


ENV INPUT_FOLDER="/input" \
    OUTPUT_FOLDER="/output"

ENV HOME="/home/${SC_USER_NAME}"
WORKDIR ${HOME}


############################################################
## MRtrix(3)
RUN apt-get -qq update \
    && apt-get install -yq --no-install-recommends \
    curl \
    dc \
    libeigen3-dev \
    libfftw3-dev \
    libgl1-mesa-dev \
    libpng-dev \
    libqt5opengl5-dev \
    libqt5svg5-dev \
    libtiff5-dev \
    qt5-default \
    zlib1g-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR ${HOME}/mrtrix3
RUN git clone -b "master" --depth 1 https://github.com/MRtrix3/mrtrix3.git . \
    && ./configure "" && NUMBER_OF_PROCESSORS=4 ./build -persistent -nopaginate \
    && rm -rf tmp

WORKDIR ${HOME}/art
RUN curl -fsSL https://osf.io/73h5s/download \
    | tar xz --strip-components 1

WORKDIR ${HOME}/ants
RUN curl -fsSL https://osf.io/yswa4/download \
    | tar xz --strip-components 1

ENV ANTSPATH="$HOME/ants/bin" \
    ARTHOME="$HOME/art" \
    PATH="$HOME/mrtrix3/bin:$HOME/ants/bin:$HOME/art/bin:$PATH"

############################################################
## Freesurfer
WORKDIR ${HOME}
RUN apt-get update && apt-get install -y tcsh bc libgomp1 perl-modules \
    && rm -rf /var/lib/apt/lists/*
RUN wget -N -qO- ftp://surfer.nmr.mgh.harvard.edu/pub/dist/freesurfer/6.0.0/freesurfer-Linux-centos6_x86_64-stable-pub-v6.0.0.tar.gz | tar -xzv -C ${HOME} 
# \ && rm -rf ${HOME}/freesurfer/subjects  # we actually need subjects/fsaverage for recon-all
ENV FREESURFER_HOME ${HOME}/freesurfer
COPY --chown=${SC_USER_NAME}:${SC_USER_NAME} docker/alpine/freesurfer_license.txt ${FREESURFER_HOME}/license.txt
ENV FSFAST_HOME==$FREESURFER_HOME/fsfast \
    MINC_BIN_DIR=$FREESURFER_HOME/mni/bin \
    MNI_DIR=$FREESURFER_HOME/mni \
    PERL5LIB=$FREESURFER_HOME/mni/share/perl5
ENV PATH=$FREESURFER_HOME/bin:$MINC_BIN_DIR:$PATH

############################################################
## FSL
WORKDIR ${HOME}
ENV FSLDIR ${HOME}/fsl
RUN wget https://fsl.fmrib.ox.ac.uk/fsldownloads/fslinstaller.py &&\ 
    echo "" | python fslinstaller.py -d ${FSLDIR} &&\ 
    . ${FSLDIR}/etc/fslconf/fsl.sh
ENV FSLOUTPUTTYPE="NIFTI_GZ" \
    FSLTCLSH="$FSLDIR/bin/fsltclsh" \
    FSLWISH="$FSLDIR/bin/fslwish" \
    LD_LIBRARY_PATH=$FSLDIR/fslpython/envs/fslpython/lib/ \
    LD_LIBRARY_PATH="$FSLDIR/lib:$LD_LIBRARY_PATH" \
    PATH=$FSLDIR/share/fsl/bin:$PATH

#############################################################################
## non-containerized Synb0-Disco
# 1: clone the github repo
WORKDIR ${HOME}
## install PyTorch for Synb0-Disco
RUN .venv/bin/pip --no-cache install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu &&\
    ## clone synb0-disco
    mkdir synb0-disco && git clone -b "master" --depth 1 https://github.com/MASILab/Synb0-DISCO ${HOME}/synb0-disco &&\
    rm -rf ${HOME}/synb0-disco/v1_0 &&\
    # save a bit of space; only 430MB and most of it is the Neural Net save files (75MB each * 5 folds)
    ### create symbolic links for other .sh files used by synb0-disco
    ln -s ${HOME}/synb0-disco/data_processing/normalize_T1.sh /usr/local/bin &&\
    ## pre-create INPUTS / OUTPUTS directories in synb0-disco; set all permissions
    mkdir synb0-disco/INPUTS &&\
    chmod gua+rwx synb0-disco/INPUTS &&\
    mkdir synb0-disco/OUTPUTS &&\
    chmod gua+rwx synb0-disco/OUTPUTS

# 2: overwrite pipeline.sh with the correct paths in our system
ENV PIPELINE_PATH=${HOME}/synb0-disco/src
COPY --chown=${SC_USER_NAME}:${SC_USER_NAME} docker/alpine/pipeline_synb0_disco.sh ${PIPELINE_PATH}/pipeline_no_docker.sh

# 3: make "synb0-disco" a recognized command for the bash console
### create a symbolic link and make it executable
RUN mkdir -p /usr/local/bin && \
    ln -s -f ${PIPELINE_PATH}/pipeline_no_docker.sh /usr/local/bin &&\
    mv /usr/local/bin/pipeline_no_docker.sh /usr/local/bin/synb0-disco &&\
    chmod +x /usr/local/bin/synb0-disco && \
    ## make synb0-disco TORCH to execute in CPU (mo)
    sed -i '83s/.*/    device = torch.device("cpu")/' $HOME/synb0-disco/src/inference.py &&\
    sed -i '87s/.*/    model.load_state_dict(torch.load(model_path, map_location=torch.device("cpu")))/' $HOME/synb0-disco/src/inference.py

## install ANTS & c3d for synb0-disco
RUN curl -SL https://github.com/ANTsX/ANTs/releases/download/v2.4.4/ants-2.4.4-ubuntu-20.04-X64-gcc.zip -o ./ants-2-4-4.zip &&\
    unzip ./ants-2-4-4.zip &&\
    rm -rf ./ants-2-4-4.zip && \ 
    curl -SL https://sourceforge.net/projects/c3d/files/c3d/1.0.0/c3d-1.0.0-Linux-x86_64.tar.gz/download | tar xz
ENV PATH=$PATH:$HOME/ants-2.4.4/bin/ 
ENV ANTSPATH=$HOME/ants-2.4.4/bin/
ENV PATH=$PATH:$HOME/c3d-1.0.0-Linux-x86_64/bin/ 

### Temporarily removed for GitHub building space issues
# ############################################################
# ## Spinal Cord Toolbox (command line)
# # RUN apt update && apt-get install -y curl   ## already installed for MRTrix3 
# WORKDIR ${HOME}
# RUN curl --location https://github.com/neuropoly/spinalcordtoolbox/archive/4.2.1.tar.gz | gunzip | tar x &&\
#   cd spinalcordtoolbox-4.2.1 && (yes "y" 2>/dev/null || true) | ./install_sct && cd - && rm -rf spinalcordtoolbox-4.2.1

############################################################
## python packages in requirements.in
## before pip install fsleyes, we need to install wxPython:
WORKDIR ${HOME}
RUN .venv/bin/pip --no-cache install -f https://extras.wxpython.org/wxPython4/extras/linux/gtk3/ubuntu-20.04 wxpython &&\
    .venv/bin/pip install attrdict

# COPY --chown=$NB_UID:$NB_GID requirements.in requirements.in
RUN .venv/bin/pip --no-cache install pip-tools &&\
    ## rename the previously existing "requirements.txt" from the jupyter-math service (we want to keep it for user reference)
    mv requirements.txt requirements_base_math.txt   &&\
    .venv/bin/pip-compile --build-isolation --output-file requirements.txt requirements.in   &&\
    .venv/bin/pip --no-cache install -r requirements.txt && \
    rm requirements.in && \
    echo "Your environment contains these python packages:" && \
    .venv/bin/pip list 

# copy docker bootup scripts
COPY --chown=${SC_USER_NAME}:${SC_USER_NAME} docker/alpine/*.sh docker/
# copy simcore service cli
COPY --chown=${SC_USER_NAME}:${SC_USER_NAME} service.cli/ service.cli/
# necessary to be able to call run directly without sh in front
ENV PATH="/home/${SC_USER_NAME}/service.cli:${PATH}"

# copy binaries from build
COPY --from=build --chown=${SC_USER_NAME}:${SC_USER_NAME} /build/bin comp-medimproc

# ------------------------------------------------------------------------------------
#TODO:
# uncomment and provide a healtchecker if possible
# HEALTHCHECK --interval=30s \
#             --timeout=120s \
#             --start-period=30s \
#             --retries=3 \
#             CMD ["healthchecker app"]
# ------------------------------------------------------------------------------------

ENTRYPOINT [ "/bin/sh", "docker/entrypoint.sh", "/bin/sh", "-c" ]
CMD ["run"]
