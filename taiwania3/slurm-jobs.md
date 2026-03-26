# SLURM Job Submission on Taiwania3

## CPU Job Script Template

```bash
#!/usr/bin/sh
#SBATCH -A MST109178          # Project number (required)
#SBATCH -J Job_name           # Job name
#SBATCH -p ngs53G             # Partition - must match mem and cores
#SBATCH -c 8                  # Cores (see partition table)
#SBATCH --mem=53g             # Memory (see partition table)
#SBATCH -o out.log            # Stdout file
#SBATCH -e err.log            # Stderr file
#SBATCH --mail-user=XXX@niar.org.tw
#SBATCH --mail-type=BEGIN,END

module load biology
module load BWA/0.7.17
bwa mem ...
```

## GPU Job Script Template

```bash
#!/usr/bin/sh
#SBATCH -A MST109178
#SBATCH -J Job_name
#SBATCH -p ngs1gpu            # GPU partition
#SBATCH -c 6                  # 6 CPU per GPU
#SBATCH --mem=90g
#SBATCH --gres=gpu:1          # Number of GPUs
#SBATCH -o out.log
#SBATCH -e err.log
#SBATCH --mail-user=XXX@niar.org.tw
#SBATCH --mail-type=BEGIN,END

nvidia-smi
# Your GPU commands here
```

## Submission

```bash
sbatch jobscript.sh
```

## Inline Submission (no script file)

```bash
sbatch -A MST109178 -J Job_name -p ngs53G -c 8 --mem=53g \
  -o out.log -e err.log --wrap="your_command_here"
```

## Pass External Variables

```bash
A=5
b='test'
sbatch --export=A=$A,b=$b job.sh
```

## Array Jobs

```bash
#SBATCH --array=1-10
echo $SLURM_ARRAY_TASK_ID    # Index for each task
```

Docs: https://slurm.schedmd.com/job_array.html

## Multi-node Jobs

For `ngs224core`: `-c 56 -n 4` (no need to specify `--mem`)
For `ngs448core`: `-c 56 -n 8`

## Job Management

```bash
squeue -u $USER              # List your jobs
scontrol show job $JOBID     # Job details
scancel -i $JOBID            # Cancel a job
```

## Key Rules

1. **Partition parameters must match exactly** - memory and core counts must follow the partition table
2. **Always use project account** `-A MST109178`
3. **Never compute on login nodes** - always submit via SLURM
4. **GPU/large-memory partitions cost more** - choose carefully
