# Distributed Video Encoding System

This is a distributed video processing system with fault tolerance, leader election, and automatic scaling capabilities.

## Key Features

- **Fault-Tolerant Architecture**
    - Master failover with Raft-inspired leader election
    - Worker health monitoring and automatic task reassignment
    - Data replication to backup nodes
    - Automatic restoration from backups

- **Video Processing**
    - Multi-format support (MP4, MKV, WebM, MOV)
    - Keyframe-aware video segmentation
    - Parallel shard processing with FFmpeg

- **System Intelligence**
    - Resource-aware load balancing using server scoring
    - Dynamic node discovery and registration
    - Real-time system metrics monitoring (CPU, memory, network)
    - Automatic node recovery and reconnection

- **Production-Ready Features**
    - gRPC streaming for large file transfers
    - Atomic file operations to prevent corruption
    - Graceful shutdown procedures
    - Comprehensive logging and status tracking

## System Architecture
The system consists of four main components:

1. **Client**: The client is the user's entry point to the system. It's used to upload videos for processing, retrieve the final processed videos, and check the status of ongoing tasks.

2. **Master Node**: The master node is the "brain" of the operation. It's responsible for:
   - Receiving video uploads from the client.
   - Segmenting the video into smaller shards.
   - Distributing these shards to the available worker nodes.
   - Tracking the status of each shard.
   - Retrieving the processed shards from the workers.
   - Concatenating the processed shards into the final video.
   - Handling client requests for video retrieval and status updates.

3. **Worker Nodes**: These are the workhorses of the system. Each worker node receives shards from the master, processes them (e.g., resizing, transcoding), and sends them back to the master. They also report their health and resource scores to the master.

4. **Backup Nodes**: These nodes store replicas of the video data to ensure that no data is lost if the master node fails.


## How it Works: A Step-by-Step Example

1. **Upload**  
   A user runs `client.py` to upload a video to the master node. The client streams the video in chunks to the master.

2. **Segmentation**  
   The master receives the video and uses **FFmpeg** to segment it into smaller, manageable shards based on keyframes.

3. **Distribution**  
   The master distributes these shards to available worker nodes using a load-balancing strategy based on each worker’s resource score.

4. **Processing**  
   Each worker node receives a shard, processes it with **FFmpeg** (e.g., changing the resolution or format), and notifies the master when finished.

5. **Retrieval**  
   The master retrieves the processed shards from the worker nodes.

6. **Concatenation**  
   After all processed shards are retrieved, the master uses **FFmpeg** to concatenate them into the final processed video.

7. **Download**  
   The user runs `client.py` again to download the final processed video from the master.


## Fault Tolerance in Action

### If the Master Fails:
- Worker nodes detect that the master is unavailable via health checks.  
- A **leader election** process is initiated to choose a new master from among the workers.  
- The node with the **best resource score** is typically elected as the new master.  
- The new master restores the system state from the **backup nodes** and resumes video processing tasks from where they left off.  

### If a Worker Fails:
- The master node detects that a worker is unresponsive.  
- It marks the shards assigned to that worker as **"failed."**  
- The master redistributes those failed shards to other healthy worker nodes in the cluster.  


## Prerequisites
- Python 3.9+
- FFmpeg 4.3+ with libx264
- MKVToolNix for container operations
- x264 codec libraries
- Disk space: Minimum 1GB free (SSD recommended)

## Quick Start

1. **Set Up Environment**

```

# Install system dependencies

sudo apt-get install -y ffmpeg mkvtoolnix libx264-dev
This installs the FFmpeg library. This step is different depending on the device. For Mac devices a simple `brew install ffmpeg` should suffice but for Windows, you need to install the binary from the FFmpeg site.

# Create and activate virtual environment

python -m venv venv
source venv/bin/activate

# Install Python dependencies

pip install grpcio protobuf grpcio-tools tqdm psutil ffmpeg-python
```

2. **Generate Protobuf Files**

```
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. replication.proto
```

3. **Create Test Video**

```
 create_test_video.sh  # Generates sample videos
```

## Running the System

### Start Master Node

```
 start_master.sh  # Default: localhost:50051
```

### Start Worker Nodes

```
 # Remove the backup-servers flag to run without backups, run each on a separated terminal or system
python node.py --role worker --host localhost --port 50061 --master localhost:50051 --backup-servers localhost:50061

python node.py --role worker --host localhost --port 50062 --master localhost:50051 --backup-servers localhost:50061

python node.py --role worker --host localhost --port 50063 --master localhost:50051 --backup-servers localhost:500614
```

## Client Operations

### Upload and Process Video

```
python client.py --master localhost:50051
  --upload ./test_video.mp4
  --width 640 --height 480
  --upscale-width 1920 --upscale-height 1080
  --format mkv
  --output ./processed.mkv
```

### Retrieve Processed Video

```
python client.py --master localhost:50051
  --retrieve <video_id>
  --output ./downloaded.mkv
```

### Check Processing Status

```
python client.py --master localhost:50051 --status <video_id>
```

### Advanced Options

| Parameter          | Description                                  | Default |
| ------------------ | -------------------------------------------- | ------- |
| `--poll-interval`  | Status check frequency (seconds)             | 5       |
| `--poll-timeout`   | Maximum processing wait time (seconds)       | 600     |
| `--upscale-width`  | Final output width override                  | -       |
| `--upscale-height` | Final output height override                 | -       |
| `--format`         | Output container format (mp4/mkv/webm/mov)   | mp4     |

## Helper Scripts

| Script                 | Purpose                                      |
| ---------------------- | -------------------------------------------- |
| `setup_env.sh`         | Full system setup and dependency install     |
| `start_master.sh`      | Launch master node with backup configuration |
| `start_worker.sh`      | Start 3 worker instances                     |
| `create_test_video.sh` | Generate test videos of various sizes        |
| `killscript.sh`        | Graceful system shutdown and cleanup         |

## Monitoring and Metrics

Metrics include:

- CPU/Memory utilization
- Network I/O
- Disk space
- Active tasks
- Election status
- Node scores

## Fault Tolerance Scenarios

1. **Master Failure**
   
      - Workers detect master timeout
   
      - Leader election initiated
   
      - New master restores from backups
   
      - Processing resumes automatically
   

3. **Worker Failure**
   
      - Master redistributes failed shards
   
      - Failed worker removed from pool
   
      - New workers automatically registered
   

5. **Network Partitions**
   
      - Majority quorum maintained
   
      - Split-brain prevention via term validation
   
      - Automatic reconciliation post-recovery
   

## Performance Considerations

- **Shard Size**: Optimal 10-second segments
- **Parallelism**: 3 workers recommended per 4-core CPU
- **Network**: 1Gbps+ recommended for HD content
- **Storage**: Separate disks for shards and final output
