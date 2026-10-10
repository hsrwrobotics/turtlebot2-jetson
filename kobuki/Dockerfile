# syntax=docker/dockerfile:1
FROM ros:humble-ros-base-jammy AS builder

# Pinned commit of CollaborativeRoboticsLab/kobuki (its submodules are pinned upstream).
# Bump this to pull upstream changes; changing it also invalidates the build cache.
ARG KOBUKI_REF=ed6792fbad9abb9ce6e4fb92d8f9cdcba6d15c9d

# Only kobuki_ros (kobuki_node & co.) and cmd_vel_mux have no Humble release.
# kobuki_core, kobuki_ros_interfaces, kobuki_velocity_smoother and the ecl/sophus
# libraries come from the ROS apt repo, so their submodules stay uninitialized.
WORKDIR /kobuki/src
RUN git clone https://github.com/CollaborativeRoboticsLab/kobuki.git && \
    git -C kobuki checkout --quiet ${KOBUKI_REF} && \
    git -C kobuki submodule update --init kobuki_ros cmd_vel_mux

WORKDIR /kobuki
RUN apt-get update && \
    rosdep update --rosdistro "$ROS_DISTRO" && \
    rosdep install --from-paths src --ignore-src -y \
        --dependency-types build --dependency-types buildtool --dependency-types build_export

RUN . "/opt/ros/$ROS_DISTRO/setup.sh" && \
    colcon build --cmake-args -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=OFF

FROM ros:humble-ros-base-jammy

ENV WORKSPACE_ROOT=/kobuki
ENV RMW_IMPLEMENTATION=rmw_cyclonedds_cpp

# Runtime deps resolved by rosdep from the built packages' package.xml files, plus
# what the launch files use without declaring it (joy) and the extra services.
RUN --mount=type=bind,from=builder,source=/kobuki/src,target=/tmp/src \
    echo 'APT::Install-Recommends "false";' > /etc/apt/apt.conf.d/99no-recommends && \
    apt-get update && \
    rosdep update --rosdistro "$ROS_DISTRO" && \
    rosdep install --from-paths /tmp/src --ignore-src -y --dependency-types exec && \
    apt-get install -y \
        ros-humble-joy \
        ros-humble-rmw-cyclonedds-cpp \
        ros-humble-rosbridge-server && \
    rm -rf /var/lib/apt/lists/* /root/.ros/rosdep

COPY --from=builder /kobuki/install ${WORKSPACE_ROOT}/install

RUN printf 'source /opt/ros/%s/setup.bash\nsource %s/install/setup.bash\n' \
    "$ROS_DISTRO" "$WORKSPACE_ROOT" >> /root/.bashrc

COPY --chmod=755 kobuki/entrypoint.sh /workspace_entrypoint.sh

ENTRYPOINT ["/workspace_entrypoint.sh"]
WORKDIR ${WORKSPACE_ROOT}
