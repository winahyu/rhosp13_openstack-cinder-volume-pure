# rhosp13 openstack-cinder-volume-pure creation

### additional referrence : https://github.com/PureStorage-OpenConnect/tripleo-deployment-configs

```bash
mkdir docker-pure
cd docker-pure
touch .dockerignore
mkdir licenses
curl -o licenses/LICENSE "https://raw.githubusercontent.com/PureStorage-OpenConnect/tripleo-deployment-configs/master/RHOSP13/licenses/LICENSE"

### prepare the Dockerfile

cat > Dockerfile <<EOF
FROM registry.access.redhat.com/rhosp13/openstack-cinder-volume
MAINTAINER Pure Storage, Inc.
LABEL name="rhosp13/openstack-cinder-volume-pure" vendor="Pure Storage" version="1.0" release="13" summary="Red Hat OpenStack Platform 13.0 cinder-volume Pure Storage FlashArray" description="Cinder plugin for Pure Storage FlashArray"
USER root
RUN yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
RUN yum install python-pip -y
RUN /usr/bin/pip install purestorage
# Create a default multipath.conf file
RUN mpathconf --enable
# Add required license as text file in Liceses directory (GPL, MIT, APACHE, Partner End User Agreement, etc)
COPY licenses /licenses
# switch the container back to the default user
USER cinder
EOF


docker build .  --> get the docker_ID at the end of successfuill build
docker tag <docker_ID> <undercloud_ip>:8787/rhosp13/openstack-cinder-volume-pure:latest
docker push <undercloud_ip>:8787/rhosp13/openstack-cinder-volume-pure

cd ./~

### update templates/environments/overcloud_images.yaml  --> replace DockerCinderVolumeImage: parameter from openstack-cinder-volume to openstack-cinder-volume-pure:latest

### prepare templates/cinder-pure.yaml 

cat > templates/cinder-pure.yaml <<EOF
# A Heat environment file which can be used to enable a
# Cinder Pure Storage FlashArray iSCSI backend, configured via puppet
resource_registry:
  OS::TripleO::Services::CinderBackendPure: /usr/share/openstack-tripleo-heat-templates/puppet/services/cinder-backend-pure.yaml
  OS::TripleO::NodeExtraConfigPost: /home/stack/templates/pure-temp.yaml

parameter_defaults:
  CinderEnableIscsiBackend: false
  CinderEnablePureBackend: true
  CinderPureBackendName: 'tripleo_pure'
  CinderPureStorageProtocol: 'FC'
  CinderPureSanIp: '<PURE_STORAGE_MGMT_API>'
  CinderPureAPIToken: '<PURE_STORAGE_API_TOKEN>'
  CinderPureUseChap: false
  CinderPureMultipathXfer: true
  CinderPureImageCache: true
EOF

=====


### prepare templates/pure-temp.yaml

cat > templates/pure-temp.yaml <<EOF
heat_template_version: queens

description: Sets up MPIO and udev rules on all nodes

parameters:
  servers:
    type: json

resources:
    PureSetup:
      type: OS::Heat::SoftwareConfig
      properties:
        group: script
        config: |
          #!/bin/bash
          sudo mpathconf --enable
          sudo systemctl start multipathd
          sudo systemctl enable multipathd
          cat <<EOF >>/tmp/99-pure-storage.rules
          ACTION=="add|change", KERNEL=="sd*[!0-9]", SUBSYSTEM=="block", ENV{ID_VENDOR}=="PURE", ATTR{queue/max_sectors_kb}="4096"
          # Use noop scheduler for high-performance solid-state storage
          ACTION=="add|change", KERNEL=="sd*[!0-9]", SUBSYSTEM=="block", ENV{ID_VENDOR}=="PURE", ATTR{queue/scheduler}="noop"
          # Reduce CPU overhead due to entropy collection
          ACTION=="add|change", KERNEL=="sd*[!0-9]", SUBSYSTEM=="block", ENV{ID_VENDOR}=="PURE", ATTR{queue/add_random}="0"
          # Spread CPU load by redirecting completions to originating CPU
          ACTION=="add|change", KERNEL=="sd*[!0-9]", SUBSYSTEM=="block", ENV{ID_VENDOR}=="PURE", ATTR{queue/rq_affinity}="2"
          # Set the HBA timeout to 60 seconds
          ACTION=="add", SUBSYSTEMS=="scsi", ATTRS{model}=="FlashArray      ", RUN+="/bin/sh -c 'echo 60 > /sys/\$DEVPATH/device/timeout'"
          EOF
          sudo cp /tmp/99-pure-storage.rules /etc/udev/rules.d/99-pure-storage.rules
          sudo /sbin/udevadm control --reload-rules
          sudo /sbin/udevadm trigger --type=devices --action=change

    ExtraDeployment:
      type: OS::Heat::SoftwareDeploymentGroup

      properties:
        servers: {get_param: servers}
        config: {get_resource: PureSetup}
        actions: [CREATE]
EOF
=====

### deploy with additional -e templates/cinder-pure.yaml 

```
