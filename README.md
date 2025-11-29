# VoD Demo with Ceph

![VoD Demo Teaser](docs/teaser.png)

## 🔧 MicroCeph & OSDs

MicroCeph is a lightweight Ceph deployment tool that simplifies the process of setting up a Ceph cluster. It can be used to quickly deploy a Ceph cluster with minimal configuration. OSDs (Object Storage Daemons) are the storage units in a Ceph cluster that store data. In this demo, we will create 3 loop-backed OSDs to form a basic Ceph cluster. The minimal recommended number of OSDs for a production Ceph cluster is 3 to ensure data redundancy and fault tolerance.

1. **Install MicroCeph via Snap**
```bash
sudo snap install microceph
```

2. **Initialize the Cluster**
```bash
sudo microceph init
```

3. **Create 3 loop-backed OSDs (2GB each) to allow size=3 pools later**
```bash
sudo dd if=/dev/zero of=/var/local/osd-1.img bs=1M count=2048
sudo dd if=/dev/zero of=/var/local/osd-2.img bs=1M count=2048
sudo dd if=/dev/zero of=/var/local/osd-3.img bs=1M count=2048
sudo losetup -f /var/local/osd-1.img
sudo losetup -f /var/local/osd-2.img
sudo losetup -f /var/local/osd-3.img
```

4. **Check which loop devices were assigned**
```bash
losetup -a
```

5. **Add OSDs to MicroCeph**
```bash
sudo microceph disk add /dev/loopX
sudo microceph disk add /dev/loopY
sudo microceph disk add /dev/loopZ
```
- Replace `/dev/loopX`, `/dev/loopY`, `/dev/loopZ` with the actual loop device names from the previous step.

6. **Verify OSDs & Cluster Status**
```bash
sudo microceph disk list
sudo microceph.ceph -s
```

## 🗄️ Create CephFS
Now that the OSDs are added, you can create pools (data pools & metadata pools) for your CephFS.

1. **Create CephFS Pools with PG=8**
```bash
sudo microceph.ceph osd pool create cephfs_data 8
sudo microceph.ceph osd pool create cephfs_meta 8
```

2. **Enable CephFS Application on the Pools**
```bash
sudo microceph.ceph osd pool application enable cephfs_data cephfs
sudo microceph.ceph osd pool application enable cephfs_meta cephfs
```

3. **Set Pool Size to 3 for Replication**
```bash
sudo microceph.ceph osd pool set cephfs_data size 3
sudo microceph.ceph osd pool set cephfs_meta size 3
```

4. **Create the Filesystem**
```bash
sudo microceph.ceph fs new mycephfs cephfs_meta cephfs_data
```

5. **(Optional) Enable Bulk Writes for Better Performance**
```bash
sudo microceph.ceph osd pool set cephfs_data bulk true
```

## 📂 Mount CephFS

1. **Install ceph client tools and kernel cephfs support**
```bash
sudo apt update
sudo apt install -y ceph-common
```

2. **Create mount point and keyring for client.admin (simplest for demo)**
```bash
sudo mkdir -p /mnt/cephfs
```

3. **Mount CephFS**
```bash
ADMIN_KEY=$(sudo microceph.ceph auth get-key client.admin)
sudo mount -t ceph <MON_IP>:6789:/ /mnt/cephfs -o name=admin,secret=$ADMIN_KEY
```
- Replace <MON_IP> with the IP you chose during microceph init.
- Get MON details (address, port) if needed:
  ```bash
  sudo microceph.ceph mon dump
  ```

4. **Verify Mount**
```bash
df -h /mnt/cephfs
# Test write and read
echo "hello cephfs" | sudo tee /mnt/cephfs/test.txt
cat /mnt/cephfs/test.txt
```

## 🎬 VoD Demo Setup

1. **Create directories for videos and HLS output**
```bash
sudo mkdir -p /mnt/cephfs/videos
sudo mkdir -p /mnt/cephfs/hls
sudo chown -R $USER:$USER /mnt/cephfs
```
- `videos` directory will store the source video files.
- `hls` directory will store the generated HLS segments and playlists (.m3u8 + .ts).

2. **Copy sample video to CephFS videos directory**
```bash
sudo cp sample.mp4 /mnt/cephfs/videos/
```

3. **Generate HLS segments and playlist using FFmpeg**
```bash
ffmpeg -i /mnt/cephfs/videos/sample.mp4 \
  -c:v libx264 -c:a aac -strict -2 \
  -start_number 0 -hls_time 10 -hls_list_size 0 -f hls \
  /mnt/cephfs/hls/sample.m3u8
```
This command will create video clips (`sample0.ts`, `sample1.ts`, etc.) and a playlist (`sample.m3u8`) in the `hls` directory.

4. **Configure Nginx to serve HLS content**
- Edit `/etc/nginx/sites-enabled/default` to add the following location block in the server section `server { ... }`:
```bash
sudo nano /etc/nginx/sites-enabled/default
```
```nginx
location /vod/ {
   alias /mnt/cephfs/hls/;
   autoindex on;
   add_header Cache-Control no-cache;
   types {
      application/vnd.apple.mpegurl m3u8;
      video/mp2t ts;
   }
}
```
- Then check config & restart Nginx:
```bash
sudo nginx -t
sudo systemctl restart nginx
```

5. **Set permissions for all to read HLS files**
```bash
sudo chmod -R 755 /mnt/cephfs/hls
```

6. **Access the VoD content via web browser or video player**
- Open a web browser or video player that supports HLS (like VLC).
- Navigate to `http://<SERVER_IP>/vod/sample.m3u8`
- Replace `<SERVER_IP>` with the IP address of your Nginx server.
- If running locally, you can use `http://localhost/vod/sample.m3u8`.

## 🌐 Frontend Development Using Flask

You can run our simple Flask application to serve the HLS video content.

1. **Prepare conda environment and install Flask**
```bash
conda create -n <env_name> python=3.10 -y
conda activate <env_name>
pip install flask
```
- Replace `<env_name>` with your desired environment name.

2. **Run the Flask application**
```bash
python app.py
```