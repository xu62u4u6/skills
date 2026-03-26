# Advanced Operations on Taiwania3

## 1. Storage Quota Queries

### Check /work and /home
```bash
/usr/lpp/mmfs/bin/mmlsquota -u $USER --block-size auto fs01 fs02
# fs01 = /work, fs02 = /home
```

### Check /staging/biology
```bash
/usr/lpp/mmfs/bin/mmlsquota -u $USER --block-size auto 5Kstaging:biology
```

### Check /staging/reserve
```bash
/usr/lpp/mmfs/bin/mmlsquota -j $Fileset --block-size auto 5Kstaging
# $Fileset = your directory name under /staging/reserve/
```

## 2. File Permissions & Sharing

### Share read access to someone
```bash
# Both lines required:
setfacl -R -m u:someone:r-X,g::---,o::--- /staging/reserve/$Fileset
setfacl -d -m u:someone:r-X,g::---,o::--- /staging/reserve/$Fileset
```
- Replace `r-X` with `rwX` for write access

### Grant parent directory traverse permission
```bash
folder="/path/to/folder"
someone="username"
while [ "$folder" != "/" ]; do
    setfacl -m u:${someone}:x "$folder"
    folder=$(dirname "$folder")
done
```

### Verify permissions
```bash
getfacl /staging/reserve/$Fileset
```

### Remove permissions
```bash
setfacl -x u:someone /staging/reserve/$Fileset
setfacl -d -x u:someone /staging/reserve/$Fileset
```

## 3. Job History & Memory Usage

### Check job memory usage
```bash
sacct -j $jobid -o JobID,JobName,Partition,User,NCPUS,AllocNodes,maxrss,Start,End,Elapsed,State
# MaxRSS in the .batch row = peak memory usage
```

### Query job history
```bash
sacct --starttime YYYY-MM-DD -u $USER -o JobID,JobName,Partition,State,ExitCode
```

## 4. Cluster Status Queries

### Queue status
```bash
qstat -ngs
```

### Node status (detailed)
```bash
ngsnodes
# Shows per-node CPU/GPU/memory usage and partition assignments
```

### Partition info
```bash
sinfo -s | grep ngs
scontrol show partition <partition_name>
```

### QoS limits
```bash
sacctmgr show qos -o format=name,MinTRES%28,MaxTRES%28,MaxJobsPU | grep ngs
```

## 5. File Encryption

### Encrypt files on HPC storage
```bash
# Compress & encrypt
tar -czvf - your_file_or_directory | openssl aes-256-cbc -pbkdf2 -salt -k password -e -out /path/to/file.tar.gz

# Decrypt & decompress
openssl aes-256-cbc -pbkdf2 -salt -k password -d -in /path/to/file.tar.gz | tar xzf -
```

## 6. GUI / Desktop Environment

### ThinLinc (Windows/Mac)
- Download: https://www.cendio.com/thinlinc/download/
- Hostname: `t3-c4.nchc.org.tw`, password field = OTP method (1/2/3)

### MobaXterm (Windows)
- Download: https://mobaxterm.mobatek.net/download.html
- Enable X11 Forwarding in Advanced SSH settings

### XQuartz (Mac)
- Download: https://www.xquartz.org/
- Connect: `ssh -Y username@t3-c4.nchc.org.tw`

### GUI Applications
```bash
# IGV
/opt/ohpc/Taiwania3/pkg/biology/IGV/IGV_v2.10.3/igv.sh

# RStudio
/opt/ohpc/Taiwania3/pkg/biology/RSTUDIO/rstudio_v2021.09.0-351/bin/rstudio
```

## 7. R Package Installation

```bash
module load biology R/4.2.1
R
```
```r
install.packages('ggplot2')  # Answer "yes" to use personal library
.libPaths()                  # Check library paths
```

## 8. Running R Scripts

```bash
module load biology R/4.3.3
Rscript myRcode.r
# Or use full path:
# /work/opt/ohpc/Taiwania3/pkg/biology/R/R_v4.1.0/bin/Rscript myRcode.r
```

## 9. RAPIDS (GPU)

```bash
# Server side
salloc -A MST109178 -p ngs1gpu -c 6 --mem=90g --gres=gpu:1 -J Rapids_Job srun --pty bash
export SINGULARITYENV_TINI_SUBREAPER=1
ml singularity
singularity run --nv /work/opt/ohpc/Taiwania3/pkg/biology/Rapids/rapids_cuda11.0/rapidsai_21.10-cuda11.0-runtime-ubuntu18.04.sif
# Note the port number (e.g. 8888) and node name (e.g. gpn3002)

# Client side - SSH tunnel
ssh -NfL {localport}:{nodename}:{port} {username}@t3-c4.nchc.org.tw
# Then open localhost:{localport} in browser

# IMPORTANT: Cancel job when done
scancel -i $jobid
```

## 10. SRAtoolkit First-Time Setup

```bash
/opt/ohpc/Taiwania3/pkg/biology/SRAToolkit/sratoolkit_v2.11.1/bin/vdb-config --interactive
# Press 's' to save, 'x' to exit
```

## 11. Cloud Storage (Google Drive, AWS, etc.)

```bash
module load biology
module load Rclone/1.59.0-DEV
# Configure: https://rclone.org/docs/
```

## 12. Specialized Software

- **Parabricks**: See dedicated guide (T3生醫節點 Parabricks使用說明)
- **AlphaFold v2**: See dedicated guide (台灣杉三號生醫服務平台AlphaFold2說明)
