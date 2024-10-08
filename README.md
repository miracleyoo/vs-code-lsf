# vs-code-lsf
This repo contains code and instructions on how to run [Visual Studio Code](https://code.visualstudio.com/) localy with all code executed on remote LSF cluster. It was inspired by discussion [here](https://github.com/microsoft/vscode-remote-release/issues/1722#issuecomment-1216040876).

# How to use
You'll need Visual Studio Code with installed `Remote` extension and ssh access to the head node of lsf cluster (it'll be referenced as `3dra2` below) through vpn or ssh tunnel if necessary.
The idea is to run `sshd` server as lsf job and connect to it from Visual Studio using Remote SSH through the tunnel.
## Remote server (LSF)
Login to the head node, download bsub script and run it:
```
wget https://raw.githubusercontent.com/iaaka/vs-code-lsf/main/bsub.vscode.tunnel.sh
./bsub.vscode.tunnel.sh
```
The script should start the job nammed `vs-code-tunnel`. The script will not start the job if one is already running and it will wait untill job is started.
Memory, number of cores, queue and gpu memory can be specified as command line arguments (defaults are `40000`, `4`, `normal`, and `6000`). GPU resources only requested if queue name contains `gpu`.

## Local machine
First edit your `~/.ssh/config` by adding following lines:
```
Host 3dra2comp
  ProxyCommand ssh 3dra2 "nc \$(/pkg/qct/software/platform/lsf/10.1/GPUSLASA/10.1/linux3.10-glibc2.17-x86_64/bin/bjobs -o first_host -J vs-code-tunnel -noheader) 5678"
  StrictHostKeyChecking no
  User <USER>
```
Replace  `<USER>` with your 3dra2 user name. Thanks to `ProxyCommand` this will allow to determine host name by job name (note `vs-code-tunnel` matches job name from bsub script.)

The command `/pkg/qct/software/platform/lsf/10.1/GPUSLASA/10.1/linux3.10-glibc2.17-x86_64/bin/bjobs` can be located on your own server by `which bjobs`, and please replace the command above with your own bjobs path.

You can check whether it works by 
```
ssh 3dra2comp
```
If everything is allright it should send you to the compute node.
Now launch Visual Studio, go to Remote SSH extension, chose 3dra2comp and voila!
## Cleanup
It is better to kill job when you stop working with Visual Studio to release resources. You can do it by:
```
bkill $(bjobs  -o JOBID -J vs-code-tunnel -noheader)
```

# Future development
1. Make lsf job start on Visual Studio connect (I have tried `RemoteCommand` in ssh config, but so far unsuccesfully)
2. Other option can be to make local Visual Studio launcher that first starts the job and then launches Visual Studio. Probably [this](https://scicomp.ethz.ch/wiki/VSCode) is relevant.
3. Add suppots for multiple sessions (with different resources) simultaneously. Possible interference of multiple sshd running on the same node? Job names needs to be diversified.  

# Alternatives
1. [code-server](https://github.com/coder/code-server) is server version of vs-code (opensource branch) that can be accessed through the browser. See [docker](https://hub.docker.com/r/linuxserver/code-server). In can be run on farm by running following code in interactive session:
```
/software/singularity-v3.9.0/bin/singularity exec \
  -B /nfs,/lustre,/software \
  --home /tmp/$(whoami) \
  --cleanenv \
  --env HOST=$(hostname) \
  /nfs/cellgeni/pasham/singimage/codeserver.sif \
  /bin/bash -c "export XDG_DATA_HOME=/nfs/cellgeni/pasham;export PASSWORD=ppp;/app/code-server/bin/code-server --bind-addr ${HOST}:8443"
```
and then conecting to ${HOST}:8443 (where ${HOST} is the node where job is running) from browser. However I didn't manage to make majority of important features to work (r/pyrhon graphics for example)
