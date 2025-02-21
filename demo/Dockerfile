# MTConnect Public C++ Agent Docker image build instructions

# TODO: Do we need notes like this to be in `Dockerfile`? I think the docs might be a better place for this. IMHO most notes in this file should be moved to the docs.
# ---------------------------------------------------------------------
# notes
# ---------------------------------------------------------------------
#
# to build a cross-platform image, push to docker hub, and run it -
# (see CMakeLists.txt for current version number) -
#
#   docker buildx build \
#     --platform linux/amd64,linux/arm64 \
#     --tag mtconnect/agent:2.0.0.12_RC18 \
#     --push \
#     .
#
#   To run tests, use `--build-arg WITH_TESTS=true`
#
#   # Note: In this case, I would suggest to map port `5000` to `5000`. The user can always change the port according to their needs.
#   docker run -it --rm --init --name agent -p5000:5000 \
#     mtconnect/agent:2.0.0.12_RC18
#
# then visit http://localhost:5000 to see the demo output.

# ---------------------------------------------------------------------
# os
# ---------------------------------------------------------------------

# base image - ubuntu has amd64, arm64 etc.
# 22.04 is the current long term support release, maintained until 2025-04.
FROM ubuntu:22.04 AS os

# tzinfo hangs without this
ARG DEBIAN_FRONTEND='noninteractive'

# ---------------------------------------------------------------------
# build
# ---------------------------------------------------------------------

FROM os AS build

# limit cpus so don't run out of memory on local machine
# symptom: get error - "c++: fatal error: Killed signal terminated program cc1plus"
# can turn off if building in cloud
ARG CONAN_CPU_COUNT=2

ARG WITH_RUBY='True'

# set some variables
ENV PATH="$HOME/venv3.9/bin:$PATH"
ENV CONAN_PROFILE='conan/profiles/docker'

# update os and add dependencies
# note: Dockerfiles run as root by default, so don't need sudo
RUN  apt-get update \
  && apt-get install -y \
       autoconf \
       automake \
       build-essential \
       cmake \
       git \
       python3 \
       python3-pip \
       rake \
       ruby \
  && rm -rf /var/lib/apt/lists/* \
  && pip install conan -v 'conan==2.0.9'

# make an agent directory and cd into it
WORKDIR /root/agent

# bring in the repo contents, minus .dockerignore files
COPY . .

ARG WITH_TESTS=false
ARG WITH_TESTS_ARG=argument
ARG SHARED=False

# Build and optionally test
RUN  if [ -z "$WITH_TESTS" ] || [ "$WITH_TESTS" = "false" ]; then \
       WITH_TESTS_ARG="--test-folder="; \
     else \
       WITH_TESTS_ARG=""; \
     fi \
  && conan profile detect \
  && conan create . \
       --build=missing \
       -c "tools.build:jobs=$CONAN_CPU_COUNT" \
       -o agent_prefix=mtc \
       -o cpack=True \
       -o "with_ruby=$WITH_RUBY" \
       -o cpack_destination=/root/agent \
       -o cpack_name=dist \
       -o cpack_generator=TGZ \
       -pr "$CONAN_PROFILE" \
       ${WITH_TESTS_ARG}

# ---------------------------------------------------------------------
# release
# ---------------------------------------------------------------------

FROM os AS release

# TODO: How about shortening the description to MTConnect Agent or at least MTConnect C++ Agent?
LABEL author='mtconnect' description='MTConnect C++ Agent'

ARG BIN_DIR='/usr/local/bin'
ARG LIB_DIR='/usr/local/lib'

ARG MTCONNECT_CONF_DIR='/etc/mtconnect'
ARG MTCONNECT_DATA_DIR='/usr/local/share/mtconnect'
ARG MTCONNECT_LOG_DIR='/var/log/mtconnect'

ENV MTCONNECT_CONF_DIR="$MTCONNECT_CONF_DIR"
ENV MTCONNECT_DATA_DIR="$MTCONNECT_DATA_DIR"
ENV MTCONNECT_LOG_DIR="$MTCONNECT_LOG_DIR"
ENV DEMO_DIR="$MTCONNECT_DATA_DIR/demo"

# install ruby for simulator
RUN  apt-get update \
  && apt-get install -y ruby

# change to a new non-root user for better security.
# this also adds the user to a group with the same name.
# -m creates a home folder, ie /home/<username>
ARG UID=1000
ARG GID=1000

RUN  groupadd \
       --gid $GID \
       agent \
  && useradd \
       --create-home \
       --uid $UID \
       --gid $GID \
       agent

USER agent
WORKDIR /home/agent

# install agent executable
COPY --chown=agent:agent --from=build /root/agent/dist.tar.gz /home/agent/

# Extract the data
RUN   tar xf dist.tar.gz \
   && rm dist.tar.gz

# Copy the agent binary and create folders used by the agent
USER root
RUN  mkdir -p "$BIN_DIR" "$LIB_DIR" \
  && cp /home/agent/dist/bin/* "$BIN_DIR" \
  && cp /home/agent/dist/lib/* "$LIB_DIR" \
  && mkdir -p "$MTCONNECT_CONF_DIR" \
              "$MTCONNECT_DATA_DIR" \
              "$MTCONNECT_LOG_DIR" \
  && chown agent:agent "$MTCONNECT_CONF_DIR" \
              "$MTCONNECT_DATA_DIR" \
              "$MTCONNECT_LOG_DIR"

USER agent

# Copy the agent data
RUN cp -r /home/agent/dist/share/mtconnect/schemas \
          /home/agent/dist/share/mtconnect/simulator \
          /home/agent/dist/share/mtconnect/styles \
          /home/agent/dist/share/mtconnect/demo \
          "$MTCONNECT_DATA_DIR" \
  && cp /home/agent/dist/share/mtconnect/demo/agent/agent.dock "$MTCONNECT_CONF_DIR/agent.cfg" \
  && cp /home/agent/dist/share/mtconnect/demo/agent/Devices.xml "$MTCONNECT_CONF_DIR" \
  && rm -r /home/agent/dist

# expose port
EXPOSE 5000

# default command - can override with docker run or docker-compose command.
# this runs the adapter simulator and the agent using the sample config file.
# note: must use shell form here instead of exec form, since we're running
# multiple statements using shell commands (& and &&).
# see https://stackoverflow.com/questions/46797348/docker-cmd-exec-form-for-multiple-command-execution
CMD /usr/bin/ruby "$MTCONNECT_DATA_DIR/simulator/run_scenario.rb" -p 7879 -l "$DEMO_DIR/agent/mazak.txt" \
  & /usr/bin/ruby "$MTCONNECT_DATA_DIR/simulator/run_scenario.rb" -p 7878 -l "$DEMO_DIR/agent/okuma.txt" \
  & mtcagent run

# ---------------------------------------------------------------------
# note
# ---------------------------------------------------------------------

# after setup, the dirs look like this -
#
# /usr/local/bin
#  |-- agent - the cppagent application
#
# /etc/mtconnect - Configuration files agent.cfg and Devices.xml
#
# /usr/local/share/mtconnect
#  |-- schemas - xsd files
#  |-- simulator - agent.cfg, simulator.rb, vmc-3axis.xml, log.txt
#  |-- styles - styles.xsl, styles.css, favicon.ico, etc
#
# /home/agent - the user's directory
#
# /var/log/mtconnect - logging directory
