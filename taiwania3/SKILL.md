---
name: taiwania3
description: NCHC GP1 Taiwania3 biomedical HPC node usage guide - SLURM job submission, partition/queue selection, storage management, module loading, GPU usage, and system administration for Taiwan's national supercomputer biomedical platform.
version: 1.0.0
allowed-tools: Read, Grep, Glob, Bash
---

# 國網中心GP1生醫核心設施 - 台灣杉三號生醫專用節點使用指南

Reference guide for NCHC Taiwania3 biomedical dedicated nodes. Use this skill when the user needs help with job submission, partition selection, storage queries, module loading, or any HPC operations on the Taiwania3 platform.

## When to Use This Skill

- Writing or debugging SLURM job scripts for Taiwania3
- Choosing the correct partition (queue) based on memory/CPU/GPU needs
- Managing storage space (home, work, staging)
- Loading software modules
- Setting up GPU jobs (RAPIDS, Parabricks, AlphaFold)
- Data transfer (sftp/scp/rsync)
- Checking job status, node availability, or quota usage
- Setting file permissions and sharing data with collaborators

## Key Reference Files

- `reference.md` - Complete system reference (login, storage, partitions, commands)
- `slurm-jobs.md` - Job script templates and submission patterns
- `advanced-operations.md` - Quota queries, permissions, encryption, GUI environments, specialized software
