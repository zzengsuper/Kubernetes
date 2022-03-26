# Set up GlusterFS cluster and Heketi

Follow this documentation to set up GlusterFS through Hekiti interface

## Vagrant Environment

| Role         | FQDN                  | IP             | OS          | RAM  | CPU  |
| ------------ | --------------------- | -------------- | ----------- | ---- | ---- |
| Heketi Node  | heketi.example.com    | 172.16.16.200  | Ubuntu20.04 | 1G   | 1    |
| Gluster Node | gluster-1.example.com | 172.16.16.201  | Ubuntu20.04 | 2G   | 2    |
| Gluster Node | gluster-2.example.com | 1722.16.16.202 | Ubuntu20.04 | 2G   | 2    |

## Gluster Nodes Setup (on gluster-1 and gluster-2)

Attach another hard disk to VirtualBox virtual machines
### Install glusterfs

```shell
{
  apt install -y glusterfs-server
  systemctl enable --now glusterd
}
```

## Heketi Setup (on heketi node)

### Download Heketi binaries

```shell
{
  cd /tmp
  wget https://github.com/heketi/heketi/releases/download/v10.4.0/heketi-client-v10.4.0-release-10.linux.amd64.tar.gz
  tar zxf heketi*
  mv heketi/{heketi,heketi-cli} /usr/local/bin/
}
```

### Set up user account for heketi

```shell
{
  groupadd -r heketi
  useradd -r -s /sbin/nologin -g heketi heketi
  mkdir {/var/lib,/etc,/var/log}/heketi
}
```

### Create ssh passwordless access to Gluster nodes

```shell
{
  ssh-keygen -f /etc/heketi/heketi_key -t rsa -N ''
  for node in gluster-1 gluster-2; do
    ssh-copy-id -i /etc/heketi/heketi_key.pub root@$node
  done
}
```

### Configure heketi

```shell
cp /tmp/heketi/heketi.json /etc/heketi/
```

Edit /etc/heketi/heketi.json, change executor to ssh and update sshexec options as shown below
```shell
	#Change use_auth to true
    "use_auth":true
    #change admin and user key
    "admin":{
       "key": "password"
    }
    "user":{
       "key": "password"
    }
    # Change executor to ssh, delete lines after "fstab" until }
	"executor": "ssh", 

	"_sshexec_comment": "SSH username and private key file information",
	"sshexec": {
  	  "keyfile": "/etc/heketi/heketi_key", 
  	  "user": "root", 
  	  "port": "22", 
  	  "fstab": "/etc/fstab" 
	},
    #Delete kubeexct section
```

### Update permissions on heketi directories

```shell
chown -R heketi:heketi {/var/lib,/etc,/var/log}/heketi
```

### Create systemd unit file for heketi

```shell
cat <<EOF >/etc/systemd/system/heketi.service
[Unit]
Description=Heketi Server

[Service]
Type=simple
WorkingDirectory=/var/lib/heketi
EnvironmentFile=-/etc/heketi/heketi.env
User=heketi
ExecStart=/usr/local/bin/heketi --config=/etc/heketi/heketi.json
Restart=on-failure
StandardOutput=syslog
StandardError=syslog

[Install]
WantedBy=multi-user.target
EOF
```

### Enable and start heketi service

```shell
{
  systemctl daemon-reload
  systemctl enable --now heketi
}
```

### Verification that heketi is running

```shell
curl localhost:8080/hello; echo
```

### Export environment variables for heketi-cli

```shell
export HEKETI_CLI_USER=admin
export HEKETI_CLI_KEY=password
```

## Create a Gluster cluster using heketi-cli

```shell
heketi-cli cluster create
Cluster id: 866bab43e62d96e4a5ca7fb14da92d7b #Output
```

### Add nodes to the Gluster cluster 

```shell
# Create node 1
heketi-cli node add --zone 1 --cluster 866bab43e62d96e4a5ca7fb14da92d7b --management-host-name gluster-1 --storage-host-name gluster-1

# Create node 2
heketi-cli node add --zone 1 --cluster 866bab43e62d96e4a5ca7fb14da92d7b --management-host-name gluster-2 --storage-host-name gluster-2

# node list
heketi-cli node list
Id:80e30b183ca90a856ad7814952975735	Cluster:866bab43e62d96e4a5ca7fb14da92d7b #output
Id:885025415eb2aedd459cace00d11e374	Cluster:866bab43e62d96e4a5ca7fb14da92d7b #output
```

### Add devices to the nodes in the Gluster cluster

```shell
# Add device to node 1
heketi-cli device add --name=/dev/sdb --node 80e30b183ca90a856ad7814952975735

# Add device to node 2
heketi-cli device add --name=/dev/sdb --node 885025415eb2aedd459cace00d11e374
```

### Create volumes in the Gluster cluster

```shell
heketi-cli volume create --size 10 --replica 2
heketi-cli volume list
heketi-cli volume info #below is output from volume info
92067d3781b2708115045c59e490a876
Name: vol_92067d3781b2708115045c59e490a876
Size: 10
Volume Id: 92067d3781b2708115045c59e490a876
Cluster Id: 866bab43e62d96e4a5ca7fb14da92d7b
Mount: gluster-1:vol_92067d3781b2708115045c59e490a876
Mount Options: backup-volfile-servers=gluster-2
Block: false
Free Size: 0
Reserved Size: 0
Block Hosting Restriction: (none)
Block Volumes: []
Durability Type: replicate
Distribute Count: 1
Replica Count: 2
```

### Mount volumes on client

```shell
apt install -y glusterfs-client
mount -t glusterfs -o backup-volfile-servers=gluster-2 gluster-1:vol_92067d3781b2708115045c59e490a876 /mnt
```

### Test redundancy by powering of gluster nodes

```shell
echo hello > /mnt/test
```

poweroff gluster-1 or gluster-2  /mnt/test still available from the client
