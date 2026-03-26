# Taiwania3 Biomedical Node Reference

## Login & Data Transfer

- **Host**: `t3-c4.nchc.org.tw` (or `203.145.216.54`)
- **DO NOT** use Taiwania1 (incurs extra costs)
- **DO NOT** use login node CPU/GPU for computation
- Account/password: iService > 會員中心 > 會員資訊 > 主機帳號資訊
- OTP guide: https://man.twcc.ai/@TWCC-III-manual/SJwbCxzqO#取得-OTP-認證碼
- Data upload via sftp/scp/rsync to `t3-c4.nchc.org.tw`
- Login uses 2FA: select method (1=APP OTP, 2=APP PUSH, 3=Email OTP), then password, then OTP

## Storage Spaces

| Path | Quota | Notes |
|------|-------|-------|
| `/home/$USER` | 100G | Home directory |
| `/work/$USER` | 1500GB | High-speed scratch (DO NOT exceed 1500GB - overage charged 4元/GB/month) |
| `/staging/biology/$USER` | 5T | Biology staging |
| `/staging/reserve/PI_folder` | Paid | Must apply separately |
| 巨量資料儲存服務 | Paid | Must apply separately |

**Important**: HPC storage is NOT backed up. Users must backup their own data. Storage is for short-term computation use; long-term data should go to 巨量資料儲存服務.

## Compute Resources & Rates

| Node Type | Count | Cores/Node | Memory | NSTC Rate | Academic Rate |
|-----------|-------|------------|--------|-----------|---------------|
| CPU | 45 | 56 | 384G | 0.08元/core-hr | 0.24元/core-hr |
| GPU (V100) | 2 | 8GPU | 768G | 10元/core-hr | 25元/core-hr |
| 2TB Memory | 1 | 36 | 2TB | 0.56元/core-hr | 1.68元/core-hr |
| 4TB Memory | 1 | 72 | 4TB | 0.56元/core-hr | 1.68元/core-hr |
| 6TB Memory | 2 | 112 | 6TB | 0.56元/core-hr | 1.68元/core-hr |

**Warning**: GPU and large-memory nodes have higher rates.

## Partition (Queue) Reference

### CPU Partitions
| Partition | Memory | Cores | Time Limit | Max Jobs |
|-----------|--------|-------|------------|----------|
| ngsTest | 7G | 1 | 10 min | 2 |
| ngsConsole | 7G | 1 | unlimited | 2 |
| ngs7G | 7G | 1 | 48 hrs | 1000 |
| ngs13G | 13G | 2 | 48 hrs | 500 |
| ngs26G | 26G | 4 | 96 hrs | 250 |
| ngs53G | 53G | 8 | 96 hrs | 120 |
| ngs92G | 92G | 14 | unlimited | 80 |
| ngs186G | 175G | 28 | unlimited | 40 |
| ngs372G | 350G | 56 | unlimited | 20 |
| ngs224core | 184G×4 | 56×4 (`-c 56 -n 4`) | unlimited | 4 |
| ngs448core | 184G×8 | 56×8 (`-c 56 -n 8`) | unlimited | 2 |

### Large Memory Partitions
| Partition | Memory | Cores | Time Limit | Max Jobs | Node |
|-----------|--------|-------|------------|----------|------|
| ngs512G | 500G | 9 | unlimited | 8 | 2,4TB |
| ngs1T_18 | 1000G | 18 | unlimited | 4 | 2,4TB |
| ngs2T_36 | 2000G | 36 | unlimited | 2 | 2,4TB |
| ngs4T_72 | 4000G | 72 | unlimited | 1 | 4TB |
| ngs2T_28 | 2000G | 28 | unlimited | 6 | 6TB |
| ngs3T_56 | 3000G | 56 | unlimited | 4 | 6TB |
| ngs6T_112 | 6000G | 112 | unlimited | 2 | 6TB |

### GPU Partitions (V100)
| Partition | Memory | CPU/GPU | Time Limit | Max Jobs |
|-----------|--------|---------|------------|----------|
| ngs1gpu | 90G | 6/1 (`-c 6 --gres=gpu:1`) | unlimited | 4 |
| ngs2gpu | 180G | 12/2 (`-c 12 --gres=gpu:2`) | unlimited | 2 |
| ngs4gpu | 360G | 24/4 (`-c 24 --gres=gpu:4`) | unlimited | 1 |
| ngs8gpu | 720G | 48/8 (`-c 48 --gres=gpu:8`) | unlimited | 1 |
| ngs1gput | 90G | 6/1 | 48hr | 8 |
| ngs2gput | 180G | 12/2 | 48hr | 4 |
| ngs4gput | 360G | 24/4 | 24hr | 2 |
| ngs8gput | 720G | 48/8 | 24hr | 1 |

### Course Partitions
| Partition | Memory | Cores | Time Limit | Max Jobs |
|-----------|--------|-------|------------|----------|
| ngscourse | 13G | 2 | 48 hrs | 500 |
| ngscourse92G | 92G | 14 | unlimited | 80 |

## Software Modules

Software location: `/opt/ohpc/Taiwania3/pkg/biology/<software>/<version>`

```bash
module load biology          # Load biology module group
module avail                 # List all available modules
module load R/4.3.3          # Load specific module
module load BWA/0.7.17
module unload gcc/12.3.0     # Unload module
module list                  # List loaded modules
module show R/4.3.3          # Show module info
```

## Project Membership Check

```bash
get_su_balance
# Must be in project MST109178
```

## Contact

- Phone: 03-5776085#370
- Email: bio.nchc@niar.org.tw
